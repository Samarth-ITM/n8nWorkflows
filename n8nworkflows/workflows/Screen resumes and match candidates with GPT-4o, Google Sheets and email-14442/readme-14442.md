Screen resumes and match candidates with GPT-4o, Google Sheets and email

https://n8nworkflows.xyz/workflows/screen-resumes-and-match-candidates-with-gpt-4o--google-sheets-and-email-14442


# Screen resumes and match candidates with GPT-4o, Google Sheets and email

# 1. Workflow Overview

This workflow automates resume screening and candidate-to-job matching using a webhook entry point, a central GPT-4o-based orchestration agent, four specialist AI agents, a validation/code tool, structured output parsing, confidence-based routing, result storage, and email notification.

Its main use case is recruitment operations: receiving a candidate resume plus job data, generating a structured multi-factor match assessment, then routing the result either to a normal reporting path for strong/high-confidence matches or to a manual-review path for uncertain cases.

## 1.1 Input Reception

The workflow starts with a POST webhook that receives resume and job data payloads. That payload is passed into the main AI orchestrator.

## 1.2 AI Matching Orchestration

A central LangChain agent acts as the matching orchestrator. It uses a GPT-4o chat model, four specialized tool-agents, one code-based validation tool, and a structured output parser to produce a normalized candidate assessment.

## 1.3 Specialist Evaluation Layer

The orchestrator can delegate tasks to:
- Resume Parser Agent
- Skill Analysis Agent
- Experience Assessment Agent
- Cultural Fit Agent

Each specialist uses its own GPT-4o model and a focused system prompt.

## 1.4 Validation and Structured Ranking Output

A code tool validates resume metadata quality/completeness, while a structured output parser forces the orchestrator’s final answer into a fixed JSON schema including scores, confidence, strengths, concerns, recommendation, and explanation.

## 1.5 Confidence-Based Routing

An IF node checks whether the result is strong enough to follow the high-confidence path. The workflow routes to the high-confidence branch if either:
- `confidenceLevel = high`, or
- `overallScore > 75`

All other outcomes go to the low-confidence/manual review branch.

## 1.6 Persistence and Notification

Both branches prepare structured records, store them in n8n Data Tables, and send emails:
- High-confidence: analysis record + positive report email
- Low-confidence: review queue record + manual review alert email

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block receives inbound recruitment data via HTTP. It is the workflow’s only trigger and provides the raw payload consumed by the orchestration agent.

### Nodes Involved
- Receive Resume & Job Data

### Node Details

#### Receive Resume & Job Data
- **Type and technical role:** `n8n-nodes-base.webhook` — HTTP trigger node
- **Configuration choices:**
  - Accepts `POST` requests
  - Webhook path: `recruitment-analysis`
  - Response mode: `lastNode`, so the webhook response is based on the workflow’s terminal execution result
- **Key expressions or variables used:** None in the node itself
- **Input and output connections:**
  - No input; it is the entry point
  - Outputs to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Invalid or unexpected request body format
  - Very large resume payloads may increase token usage downstream
  - Because `responseMode` is `lastNode`, long-running LLM/email/storage activity may cause webhook timeout depending on deployment setup
- **Sub-workflow reference:** None

---

## Block 2 — AI Matching Orchestration Core

### Overview
This block contains the central AI decision engine. It receives the inbound payload, invokes specialized tools and models, and synthesizes a structured final assessment while explicitly avoiding autonomous hiring decisions.

### Nodes Involved
- Matching Agent (Orchestrator)
- Matching Agent Model
- Ranking Output Parser
- Validation Logic Tool

### Node Details

#### Matching Agent (Orchestrator)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — central LangChain agent orchestrator
- **Configuration choices:**
  - Prompt mode is explicitly defined
  - Main input text is `={{ $json.body }}`
  - Uses a system message instructing it to:
    - coordinate specialized sub-agents
    - provide detailed ranking explanations
    - never make autonomous hiring decisions
    - always require human review in principle
  - `hasOutputParser` is enabled
- **Key expressions or variables used:**
  - `{{ $json.body }}`
- **Input and output connections:**
  - Main input from `Receive Resume & Job Data`
  - AI language model from `Matching Agent Model`
  - AI tools:
    - `Resume Parser Agent`
    - `Skill Analysis Agent`
    - `Experience Assessment Agent`
    - `Cultural Fit Agent`
    - `Validation Logic Tool`
  - AI output parser from `Ranking Output Parser`
  - Main output to `Check Confidence Level`
- **Version-specific requirements:** Type version `3.1`; requires n8n LangChain/AI nodes compatible with this agent interface
- **Edge cases or potential failure types:**
  - If inbound body is missing or not text/JSON-serializable, prompt quality degrades
  - Tool-calling may fail if the agent cannot infer correct tool arguments
  - Output parser may reject malformed final output
  - OpenAI rate limiting, token limits, or authentication issues
  - Hallucinated fields or inconsistent score scales if prompts are not aligned with expected schema
- **Sub-workflow reference:** None; tools are internal AI tool nodes, not workflow-call subworkflows

#### Matching Agent Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — primary LLM backing the orchestrator
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as AI language model to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `1.3`; valid OpenAI credentials required
- **Edge cases or potential failure types:**
  - OpenAI credential failure
  - Model access restrictions
  - Rate limits or token quota exhaustion
- **Sub-workflow reference:** None

#### Ranking Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON output
- **Configuration choices:**
  - Manual JSON schema
  - Required fields:
    - `candidateId`
    - `overallScore`
    - `confidenceLevel`
    - `skillMatchScore`
    - `experienceScore`
    - `culturalFitScore`
    - `strengths`
    - `concerns`
    - `recommendation`
    - `requiresHumanReview`
    - `detailedExplanation`
  - `confidenceLevel` restricted to `high`, `medium`, `low`
  - `overallScore` constrained from 0 to 100
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as AI output parser to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Parser failure if the final AI response omits required fields
  - Type mismatch, e.g. string instead of number/boolean/array
  - Score outside schema bounds
- **Sub-workflow reference:** None

#### Validation Logic Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode` — executable JavaScript validation tool callable by the orchestrator
- **Configuration choices:**
  - Manual input schema expecting a top-level object containing `resumeMetadata`
  - JavaScript validates:
    - required fields: `name`, `email`, `skills`, `experience`
    - email format
    - non-negative experience
    - minimum skill count
  - Returns a JSON-formatted string containing:
    - `isValid`
    - `missingFields`
    - `qualityScore`
    - `warnings`
- **Key expressions or variables used:**
  - Uses `query` as the tool input variable
- **Input and output connections:**
  - Connected as AI tool to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Potential schema mismatch: code sets `const resumeMetadata = query;` while the declared input schema suggests `query.resumeMetadata`
  - If the agent sends `{ resumeMetadata: {...} }`, validation may incorrectly inspect the wrapper object instead of the actual metadata
  - If the tool receives malformed data, required field checks may all fail
  - Tool returns a JSON string, not a native object; the agent must interpret it correctly
- **Sub-workflow reference:** None

---

## Block 3 — Specialist Evaluation Layer

### Overview
This block provides the specialist AI tools used by the orchestrator. Each tool is focused on a specific evaluation dimension and backed by its own GPT-4o model with tailored temperature and instructions.

### Nodes Involved
- Resume Parser Agent
- Resume Parser Model
- Skill Analysis Agent
- Skill Analysis Model
- Experience Assessment Agent
- Experience Assessment Model
- Cultural Fit Agent
- Cultural Fit Model

### Node Details

#### Resume Parser Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist callable AI tool for resume structuring
- **Configuration choices:**
  - Tool input text: `={{ $fromAI('resumeData', 'The resume content to parse and structure') }}`
  - System message instructs extraction of:
    - personal details
    - education
    - work history
    - technical skills
    - soft skills
    - certifications
    - achievements
    - missing-field validation
  - `hasOutputParser` enabled
  - Tool description clearly advertises structured JSON metadata output
- **Key expressions or variables used:**
  - `$fromAI('resumeData', ...)`
- **Input and output connections:**
  - AI model input from `Resume Parser Model`
  - Tool exposed to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Poor extraction from unstructured/scanned/OCR-damaged resumes
  - Missing candidate identifiers
  - If no parser is attached despite `hasOutputParser`, output consistency depends on model behavior
- **Sub-workflow reference:** None

#### Resume Parser Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Feeds `Resume Parser Agent`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** standard OpenAI auth/rate/token issues
- **Sub-workflow reference:** None

#### Skill Analysis Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**
  - Tool input text: `={{ $fromAI('skillData', 'Candidate skills and job requirements to analyze') }}`
  - System message focuses on:
    - candidate vs job skill comparison
    - proficiency evaluation
    - skill gaps
    - transferable skills
    - detailed scoring and explanations
  - `hasOutputParser` enabled
- **Key expressions or variables used:**
  - `$fromAI('skillData', ...)`
- **Input and output connections:**
  - AI model input from `Skill Analysis Model`
  - Tool exposed to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Weak or vague job requirements reduce scoring reliability
  - Candidate skill synonyms may be under-matched unless prompts are robust
- **Sub-workflow reference:** None

#### Skill Analysis Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Feeds `Skill Analysis Agent`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** standard OpenAI auth/rate/token issues
- **Sub-workflow reference:** None

#### Experience Assessment Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**
  - Tool input text: `={{ $fromAI('experienceData', 'Candidate work history and job experience requirements to evaluate') }}`
  - System prompt evaluates:
    - years of relevant experience
    - career progression
    - responsibility alignment
    - industry fit
    - leadership
    - impact/achievement relevance
  - `hasOutputParser` enabled
- **Key expressions or variables used:**
  - `$fromAI('experienceData', ...)`
- **Input and output connections:**
  - AI model input from `Experience Assessment Model`
  - Tool exposed to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Nonlinear careers may be interpreted inconsistently
  - Date parsing ambiguity can distort years-of-experience scoring
- **Sub-workflow reference:** None

#### Experience Assessment Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Feeds `Experience Assessment Agent`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** standard OpenAI auth/rate/token issues
- **Sub-workflow reference:** None

#### Cultural Fit Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**
  - Tool input text: `={{ $fromAI('culturalData', 'Candidate soft skills and company culture requirements to analyze') }}`
  - System message assesses:
    - communication style
    - collaboration
    - adaptability
    - problem-solving
    - leadership style
    - values alignment
  - Includes advisory warning that final cultural fit must be assessed by humans
  - `hasOutputParser` enabled
- **Key expressions or variables used:**
  - `$fromAI('culturalData', ...)`
- **Input and output connections:**
  - AI model input from `Cultural Fit Model`
  - Tool exposed to `Matching Agent (Orchestrator)`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Resume text may not contain enough evidence for cultural-fit scoring
  - High risk of inference bias if company culture requirements are underspecified
- **Sub-workflow reference:** None

#### Cultural Fit Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Feeds `Cultural Fit Agent`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** standard OpenAI auth/rate/token issues
- **Sub-workflow reference:** None

---

## Block 4 — Confidence Check and Routing

### Overview
This block evaluates the structured AI result and routes it into one of two operational paths: standard high-confidence processing or manual review handling.

### Nodes Involved
- Check Confidence Level

### Node Details

#### Check Confidence Level
- **Type and technical role:** `n8n-nodes-base.if` — conditional router
- **Configuration choices:**
  - OR combinator
  - Condition 1: `{{ $json.confidenceLevel }}` equals `high`
  - Condition 2: `{{ $json.overallScore }}` greater than `75`
  - Loose type validation, case-insensitive string comparison
- **Key expressions or variables used:**
  - `{{ $json.confidenceLevel }}`
  - `{{ $json.overallScore }}`
- **Input and output connections:**
  - Main input from `Matching Agent (Orchestrator)`
  - True output to `Prepare Analysis Data`
  - False output to `Prepare Low Confidence Data`
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - A `medium` confidence result with score > 75 still goes to the high-confidence path
  - Missing numeric score can break or misroute depending on coercion
  - If the parser output shape changes, conditions may evaluate incorrectly
- **Sub-workflow reference:** None

---

## Block 5 — High-Confidence Processing

### Overview
This block prepares the structured analysis record for strong matches, stores it, and sends a formatted report email to the hiring manager.

### Nodes Involved
- Prepare Analysis Data
- Store Analysis Results
- Send High Confidence Report

### Node Details

#### Prepare Analysis Data
- **Type and technical role:** `n8n-nodes-base.set` — field mapping/normalization
- **Configuration choices:**
  - Creates:
    - `candidateId`
    - `timestamp`
    - `overallScore`
    - `confidenceLevel`
    - `skillMatchScore`
    - `experienceScore`
    - `culturalFitScore`
    - `recommendation`
    - `detailedExplanation`
  - Timestamp uses `{{ $now }}`
- **Key expressions or variables used:**
  - `{{ $json.candidateId }}`
  - `{{ $json.overallScore }}`
  - `{{ $json.confidenceLevel }}`
  - `{{ $json.skillMatchScore }}`
  - `{{ $json.experienceScore }}`
  - `{{ $json.culturalFitScore }}`
  - `{{ $json.recommendation }}`
  - `{{ $json.detailedExplanation }}`
  - `{{ $now }}`
- **Input and output connections:**
  - Input from true branch of `Check Confidence Level`
  - Output to `Store Analysis Results`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Missing fields from parser/orchestrator output produce null or invalid mapped values
  - Date formatting of `$now` depends on n8n runtime representation
- **Sub-workflow reference:** None

#### Store Analysis Results
- **Type and technical role:** `n8n-nodes-base.dataTable` — stores analysis results in an n8n Data Table
- **Configuration choices:**
  - Auto-maps incoming fields to table columns
  - Uses placeholder Data Table ID for candidate analysis table
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Prepare Analysis Data`
  - Output to `Send High Confidence Report`
- **Version-specific requirements:** Type version `1.1`; requires n8n Data Tables support in the environment
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Auto-mapping can fail if destination columns do not exist or types mismatch
  - Permission/environment issues if Data Tables are unavailable
- **Sub-workflow reference:** None

#### Send High Confidence Report
- **Type and technical role:** `n8n-nodes-base.emailSend` — sends formatted HTML email
- **Configuration choices:**
  - Subject includes candidate ID
  - Uses placeholder sender and recipient email addresses
  - HTML email includes:
    - candidate overview
    - detailed scores
    - strengths
    - recommendation
    - detailed explanation
  - Attribution enabled
- **Key expressions or variables used:**
  - References `{{ $json.output.* }}` throughout HTML and subject
- **Input and output connections:**
  - Input from `Store Analysis Results`
  - No downstream node
- **Version-specific requirements:** Type version `2.1`; requires SMTP/Gmail-compatible email credentials configured in n8n
- **Edge cases or potential failure types:**
  - Likely field-reference mismatch: upstream `Store Analysis Results` receives plain fields such as `candidateId`, but email template expects `$json.output.candidateId`
  - If no `output` object exists, expressions may fail or render empty
  - Invalid recipient/sender configuration
  - HTML list rendering fails if `strengths` is absent; also `strengths` is not preserved in `Prepare Analysis Data`
- **Sub-workflow reference:** None

---

## Block 6 — Low-Confidence / Manual Review Processing

### Overview
This block handles uncertain or weaker assessments by preparing a review record, storing it in a review queue table, and sending an alert email requesting human intervention.

### Nodes Involved
- Prepare Low Confidence Data
- Store Low Confidence Cases
- Send Review Required Alert

### Node Details

#### Prepare Low Confidence Data
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Creates:
    - `candidateId`
    - `timestamp`
    - `overallScore`
    - `confidenceLevel`
    - `concerns`
    - `requiresHumanReview` = `true`
    - `reviewStatus` = `pending`
- **Key expressions or variables used:**
  - `{{ $json.candidateId }}`
  - `{{ $json.overallScore }}`
  - `{{ $json.confidenceLevel }}`
  - `{{ $json.concerns }}`
  - `{{ $now }}`
- **Input and output connections:**
  - Input from false branch of `Check Confidence Level`
  - Output to `Store Low Confidence Cases`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - `concerns` may be missing or non-array despite schema expectation
  - Candidate ID missing means review queue records become harder to trace
- **Sub-workflow reference:** None

#### Store Low Confidence Cases
- **Type and technical role:** `n8n-nodes-base.dataTable`
- **Configuration choices:**
  - Auto-maps incoming fields
  - Uses placeholder Data Table ID for review queue table
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Prepare Low Confidence Data`
  - Output to `Send Review Required Alert`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Column/type mismatch in Data Table
  - Environment may not support Data Tables
- **Sub-workflow reference:** None

#### Send Review Required Alert
- **Type and technical role:** `n8n-nodes-base.emailSend`
- **Configuration choices:**
  - Subject marks the result as review required
  - HTML body emphasizes low confidence and mandatory manual review
  - Uses placeholder sender and recipient addresses
  - Attribution enabled
- **Key expressions or variables used:**
  - References `{{ $json.output.* }}` throughout
  - Includes color logic based on `overallScore`
  - Uppercases confidence level
  - Renders list from `concerns`
- **Input and output connections:**
  - Input from `Store Low Confidence Cases`
  - No downstream node
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Same likely field-reference mismatch as high-confidence email: upstream shape appears flat, template expects `$json.output.*`
  - `concerns.map(...)` fails if `concerns` is absent or not an array
  - Sender/recipient credentials or addresses may be invalid
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Resume & Job Data | n8n-nodes-base.webhook | Receives POSTed resume and job payload |  | Matching Agent (Orchestrator) | ## How It Works<br>This workflow automates candidate screening and job matching for recruiters, HR operations teams, and talent acquisition leads. It eliminates the manual effort of parsing resumes, evaluating multi-dimensional candidate fit, and routing outcomes based on assessment confidence. Resume and job data are received via a POST webhook and passed directly to the Matching Agent Orchestrator, backed by a matching model and shared memory. The orchestrator coordinates four specialist agents in parallel: a Resume Parser Agent (structured extraction), a Skill Analysis Agent (competency mapping), an Experience Assessment Agent (seniority and relevance scoring), and a Cultural Fit Agent (organisational alignment evaluation). A Validation Logic Tool cross-checks outputs before a Ranking Output Parser produces a structured candidate ranking. Results are then checked against a confidence threshold — low-confidence cases trigger a review alert via email and are stored in Google Sheets for human follow-up, while high-confidence matches are prepared as analysis data, stored in Sheets, and distributed as a ranked report via email.<br>## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Matching Agent (Orchestrator) | @n8n/n8n-nodes-langchain.agent | Central AI orchestrator combining specialist assessments into final ranking | Receive Resume & Job Data; Matching Agent Model; Resume Parser Agent; Skill Analysis Agent; Experience Assessment Agent; Cultural Fit Agent; Validation Logic Tool; Ranking Output Parser | Check Confidence Level | ## Setup Steps<br>1. Import workflow; configure the POST webhook trigger URL for resume and job data ingestion.<br>2. Add AI model credentials to the Matching Agent Orchestrator, Resume Parser Agent, Skill Analysis Agent, Experience Assessment Agent, and Cultural Fit Agent.<br>3. Link Google Sheets credentials; set sheet IDs for Low Confidence Cases and Analysis Results tabs.<br>4. Connect email credentials to the Send Review Required Alert and Send High Confidence Report nodes.<br>5. Set confidence threshold values in the Check Confidence Level node.<br>## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Matching Agent Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model backing the orchestrator |  | Matching Agent (Orchestrator) | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Resume Parser Agent | @n8n/n8n-nodes-langchain.agentTool | Extracts structured resume metadata | Resume Parser Model | Matching Agent (Orchestrator) | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking.<br>## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Resume Parser Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for resume parsing specialist |  | Resume Parser Agent | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Skill Analysis Agent | @n8n/n8n-nodes-langchain.agentTool | Evaluates skill match against job requirements | Skill Analysis Model | Matching Agent (Orchestrator) | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking.<br>## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Skill Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for skill analysis |  | Skill Analysis Agent | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Experience Assessment Agent | @n8n/n8n-nodes-langchain.agentTool | Assesses relevance and depth of work history | Experience Assessment Model | Matching Agent (Orchestrator) | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking.<br>## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Experience Assessment Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for experience assessment |  | Experience Assessment Agent | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Cultural Fit Agent | @n8n/n8n-nodes-langchain.agentTool | Evaluates soft-skill and values alignment | Cultural Fit Model | Matching Agent (Orchestrator) | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Google Sheets with candidate tracking tabs pre-created<br>- Email account credentials (SMTP or Gmail OAuth)<br>## Use Cases<br>- Recruiters automating high-volume resume screening against structured job descriptions<br>## Customisation<br>- Extend specialist agents with domain-specific scoring rubrics for technical or executive roles<br>## Benefits<br>- Four parallel specialist agents evaluate candidates across skills, experience, and cultural fit simultaneously<br>## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking.<br>## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Cultural Fit Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for cultural fit analysis |  | Cultural Fit Agent | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Google Sheets with candidate tracking tabs pre-created<br>- Email account credentials (SMTP or Gmail OAuth)<br>## Use Cases<br>- Recruiters automating high-volume resume screening against structured job descriptions<br>## Customisation<br>- Extend specialist agents with domain-specific scoring rubrics for technical or executive roles<br>## Benefits<br>- Four parallel specialist agents evaluate candidates across skills, experience, and cultural fit simultaneously |
| Validation Logic Tool | @n8n/n8n-nodes-langchain.toolCode | Validates resume metadata completeness and quality |  | Matching Agent (Orchestrator) | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Google Sheets with candidate tracking tabs pre-created<br>- Email account credentials (SMTP or Gmail OAuth)<br>## Use Cases<br>- Recruiters automating high-volume resume screening against structured job descriptions<br>## Customisation<br>- Extend specialist agents with domain-specific scoring rubrics for technical or executive roles<br>## Benefits<br>- Four parallel specialist agents evaluate candidates across skills, experience, and cultural fit simultaneously<br>## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Ranking Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured final JSON output from orchestrator |  | Matching Agent (Orchestrator) | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Google Sheets with candidate tracking tabs pre-created<br>- Email account credentials (SMTP or Gmail OAuth)<br>## Use Cases<br>- Recruiters automating high-volume resume screening against structured job descriptions<br>## Customisation<br>- Extend specialist agents with domain-specific scoring rubrics for technical or executive roles<br>## Benefits<br>- Four parallel specialist agents evaluate candidates across skills, experience, and cultural fit simultaneously<br>## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Check Confidence Level | n8n-nodes-base.if | Routes results to high-confidence or manual-review path | Matching Agent (Orchestrator) | Prepare Analysis Data; Prepare Low Confidence Data | ## Check Confidence Level & Route Outcomes<br>**Why** — Confidence-based routing separates high-quality matches from uncertain assessments, directing low-confidence cases to human review without discarding them. |
| Prepare Analysis Data | n8n-nodes-base.set | Normalizes high-confidence output for storage | Check Confidence Level | Store Analysis Results | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |
| Store Analysis Results | n8n-nodes-base.dataTable | Stores high-confidence analysis records | Prepare Analysis Data | Send High Confidence Report | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |
| Send High Confidence Report | n8n-nodes-base.emailSend | Sends high-confidence candidate report email | Store Analysis Results |  | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |
| Prepare Low Confidence Data | n8n-nodes-base.set | Normalizes low-confidence output for review queue storage | Check Confidence Level | Store Low Confidence Cases | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |
| Store Low Confidence Cases | n8n-nodes-base.dataTable | Stores manual-review cases | Prepare Low Confidence Data | Send Review Required Alert | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |
| Send Review Required Alert | n8n-nodes-base.emailSend | Sends manual review alert email | Store Low Confidence Cases |  | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Google Sheets with candidate tracking tabs pre-created<br>- Email account credentials (SMTP or Gmail OAuth)<br>## Use Cases<br>- Recruiters automating high-volume resume screening against structured job descriptions<br>## Customisation<br>- Extend specialist agents with domain-specific scoring rubrics for technical or executive roles<br>## Benefits<br>- Four parallel specialist agents evaluate candidates across skills, experience, and cultural fit simultaneously |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Setup Steps<br>1. Import workflow; configure the POST webhook trigger URL for resume and job data ingestion.<br>2. Add AI model credentials to the Matching Agent Orchestrator, Resume Parser Agent, Skill Analysis Agent, Experience Assessment Agent, and Cultural Fit Agent.<br>3. Link Google Sheets credentials; set sheet IDs for Low Confidence Cases and Analysis Results tabs.<br>4. Connect email credentials to the Send Review Required Alert and Send High Confidence Report nodes.<br>5. Set confidence threshold values in the Check Confidence Level node. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## How It Works<br>This workflow automates candidate screening and job matching for recruiters, HR operations teams, and talent acquisition leads. It eliminates the manual effort of parsing resumes, evaluating multi-dimensional candidate fit, and routing outcomes based on assessment confidence. Resume and job data are received via a POST webhook and passed directly to the Matching Agent Orchestrator, backed by a matching model and shared memory. The orchestrator coordinates four specialist agents in parallel: a Resume Parser Agent (structured extraction), a Skill Analysis Agent (competency mapping), an Experience Assessment Agent (seniority and relevance scoring), and a Cultural Fit Agent (organisational alignment evaluation). A Validation Logic Tool cross-checks outputs before a Ranking Output Parser produces a structured candidate ranking. Results are then checked against a confidence threshold — low-confidence cases trigger a review alert via email and are stored in Google Sheets for human follow-up, while high-confidence matches are prepared as analysis data, stored in Sheets, and distributed as a ranked report via email. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Check Confidence Level & Route Outcomes<br>**Why** — Confidence-based routing separates high-quality matches from uncertain assessments, directing low-confidence cases to human review without discarding them. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Validation Logic Tool & Ranking Output Parser<br>**Why** — Cross-validates multi-agent outputs and structures them into a ranked candidate list, ensuring scoring consistency before confidence checking. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Receiving, Matching Agent Orchestrator & Specialist Agents<br>**Why** — Coordinates Resume Parsing, Skill Analysis, Experience Assessment, and Cultural Fit agents in parallel using shared memory for comprehensive, multi-dimensional candidate evaluation. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Store Results & Send Reports<br>**Why** — Both confidence paths store outputs in Google Sheets and distribute reports via email, maintaining a complete candidate record regardless of confidence outcome. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Screen resumes and match candidates with GPT-4o, Google Sheets and email**
   - Keep workflow inactive until credentials and placeholders are configured.

2. **Add the webhook trigger**
   - Add a **Webhook** node.
   - Name it **Receive Resume & Job Data**.
   - Set:
     - HTTP Method: `POST`
     - Path: `recruitment-analysis`
     - Response Mode: `Last Node`
   - This node expects inbound recruitment payloads, ideally containing resume content and job requirements in the request body.

3. **Add the main AI orchestrator**
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`).
   - Name it **Matching Agent (Orchestrator)**.
   - Set prompt type to **Define**.
   - In the main text input, use:
     - `{{ $json.body }}`
   - Enable structured output parsing.
   - Use this system behavior:
     - The agent is a recruitment matching orchestrator
     - It coordinates specialist agents
     - It provides recommendations only
     - It must not make autonomous hiring decisions
     - It synthesizes detailed explanations and confidence scores

4. **Connect webhook to orchestrator**
   - Connect **Receive Resume & Job Data** → **Matching Agent (Orchestrator)** using the main connection.

5. **Add the orchestrator model**
   - Add an **OpenAI Chat Model** node.
   - Name it **Matching Agent Model**.
   - Set:
     - Model: `gpt-4o`
     - Temperature: `0.2`
   - Attach OpenAI credentials.
   - Connect it to the orchestrator as the **AI Language Model**.

6. **Add the structured output parser**
   - Add a **Structured Output Parser** node.
   - Name it **Ranking Output Parser**.
   - Choose **Manual Schema**.
   - Define a schema with these required fields:
     - `candidateId` string
     - `overallScore` number, 0–100
     - `confidenceLevel` enum: `high`, `medium`, `low`
     - `skillMatchScore` number
     - `experienceScore` number
     - `culturalFitScore` number
     - `strengths` array of strings
     - `concerns` array of strings
     - `recommendation` string
     - `requiresHumanReview` boolean
     - `detailedExplanation` string
   - Connect it to the orchestrator as the **AI Output Parser**.

7. **Add the resume parser specialist**
   - Add an **AI Agent Tool** node.
   - Name it **Resume Parser Agent**.
   - Set text input to:
     - `{{ $fromAI('resumeData', 'The resume content to parse and structure') }}`
   - Add a system message instructing extraction of structured resume data:
     - personal details
     - education
     - work experience
     - technical skills
     - soft skills
     - certifications
     - achievements
     - missing fields
   - Enable output parsing for consistency.
   - Set a tool description explaining that it returns standardized resume metadata.

8. **Add the model for the resume parser**
   - Add an **OpenAI Chat Model** node.
   - Name it **Resume Parser Model**.
   - Set:
     - Model: `gpt-4o`
     - Temperature: `0.1`
   - Attach OpenAI credentials.
   - Connect **Resume Parser Model** → **Resume Parser Agent** as **AI Language Model**.
   - Connect **Resume Parser Agent** → **Matching Agent (Orchestrator)** as **AI Tool**.

9. **Add the skill analysis specialist**
   - Add an **AI Agent Tool** node.
   - Name it **Skill Analysis Agent**.
   - Use text input:
     - `{{ $fromAI('skillData', 'Candidate skills and job requirements to analyze') }}`
   - System message should instruct:
     - compare candidate skills to job requirements
     - assess proficiency
     - identify gaps
     - recognize transferable skills
     - produce scoring and explanations
   - Enable output parsing.
   - Add a descriptive tool description.

10. **Add the model for skill analysis**
    - Add an **OpenAI Chat Model** node.
    - Name it **Skill Analysis Model**.
    - Set:
      - Model: `gpt-4o`
      - Temperature: `0.2`
    - Connect model to tool as **AI Language Model**.
    - Connect tool to orchestrator as **AI Tool**.

11. **Add the experience assessment specialist**
    - Add an **AI Agent Tool** node.
    - Name it **Experience Assessment Agent**.
    - Use text input:
      - `{{ $fromAI('experienceData', 'Candidate work history and job experience requirements to evaluate') }}`
    - System message should evaluate:
      - years of relevant experience
      - career progression
      - role alignment
      - industry fit
      - leadership
      - achievement impact
    - Enable output parsing.
    - Add a descriptive tool description.

12. **Add the model for experience assessment**
    - Add an **OpenAI Chat Model** node.
    - Name it **Experience Assessment Model**.
    - Set:
      - Model: `gpt-4o`
      - Temperature: `0.2`
    - Connect model to tool as **AI Language Model**.
    - Connect tool to orchestrator as **AI Tool**.

13. **Add the cultural fit specialist**
    - Add an **AI Agent Tool** node.
    - Name it **Cultural Fit Agent**.
    - Use text input:
      - `{{ $fromAI('culturalData', 'Candidate soft skills and company culture requirements to analyze') }}`
    - System message should assess:
      - communication style
      - collaboration
      - adaptability
      - problem solving
      - leadership style
      - values alignment
    - Include an advisory note that final cultural-fit judgment must be done by humans.
    - Enable output parsing.
    - Add a descriptive tool description.

14. **Add the model for cultural fit**
    - Add an **OpenAI Chat Model** node.
    - Name it **Cultural Fit Model**.
    - Set:
      - Model: `gpt-4o`
      - Temperature: `0.3`
    - Connect model to tool as **AI Language Model**.
    - Connect tool to orchestrator as **AI Tool**.

15. **Add the validation code tool**
    - Add a **Code Tool** node (`toolCode`).
    - Name it **Validation Logic Tool**.
    - Enable **Specify Input Schema**.
    - Use a manual input schema expecting an object with:
      - `resumeMetadata` object
      - nested fields such as `name`, `email`, `skills`, `experience`
    - Add JavaScript logic that:
      - validates required fields
      - checks email format
      - evaluates experience sanity
      - checks skill count
      - computes `qualityScore`
      - returns a JSON string with `isValid`, `missingFields`, `qualityScore`, `warnings`
    - Connect it to the orchestrator as **AI Tool**.

16. **Recommended correction while rebuilding**
    - In the code tool, prefer:
      - `const resumeMetadata = query.resumeMetadata;`
    - This better matches the declared input schema than the original implementation, which uses `const resumeMetadata = query;`.

17. **Add the confidence routing node**
    - Add an **If** node.
    - Name it **Check Confidence Level**.
    - Configure OR logic with:
      - Condition A: `{{ $json.confidenceLevel }}` equals `high`
      - Condition B: `{{ $json.overallScore }}` greater than `75`
    - Connect **Matching Agent (Orchestrator)** → **Check Confidence Level**.

18. **Build the high-confidence branch**
    - Add a **Set** node named **Prepare Analysis Data**.
    - On the true output of the IF node, map:
      - `candidateId` ← `{{ $json.candidateId }}`
      - `timestamp` ← `{{ $now }}`
      - `overallScore` ← `{{ $json.overallScore }}`
      - `confidenceLevel` ← `{{ $json.confidenceLevel }}`
      - `skillMatchScore` ← `{{ $json.skillMatchScore }}`
      - `experienceScore` ← `{{ $json.experienceScore }}`
      - `culturalFitScore` ← `{{ $json.culturalFitScore }}`
      - `recommendation` ← `{{ $json.recommendation }}`
      - `detailedExplanation` ← `{{ $json.detailedExplanation }}`
    - Recommended improvement: also include `strengths` and `concerns` if the email should use them.

19. **Add storage for high-confidence analyses**
    - Add a **Data Table** node.
    - Name it **Store Analysis Results**.
    - Use **Auto-map input data**.
    - Select or create a Data Table for candidate analysis results.
    - Replace the placeholder table ID with your actual Data Table ID.
    - Connect **Prepare Analysis Data** → **Store Analysis Results**.

20. **Add the high-confidence email**
    - Add an **Email Send** node.
    - Name it **Send High Confidence Report**.
    - Configure:
      - To: hiring manager email
      - From: system email
      - Subject including candidate ID
      - HTML body with overview, detailed scores, strengths, recommendation, and detailed explanation
    - Connect **Store Analysis Results** → **Send High Confidence Report**.

21. **Recommended correction for the high-confidence email**
    - The original email template reads values from `{{ $json.output... }}`.
    - Unless your preceding node produces an `output` object, change expressions to flat fields like:
      - `{{ $json.candidateId }}`
      - `{{ $json.overallScore }}`
      - `{{ $json.skillMatchScore }}`
    - If you want to keep `strengths`, ensure **Prepare Analysis Data** includes it.

22. **Build the low-confidence branch**
    - Add a **Set** node named **Prepare Low Confidence Data**.
    - On the false output of the IF node, map:
      - `candidateId` ← `{{ $json.candidateId }}`
      - `timestamp` ← `{{ $now }}`
      - `overallScore` ← `{{ $json.overallScore }}`
      - `confidenceLevel` ← `{{ $json.confidenceLevel }}`
      - `concerns` ← `{{ $json.concerns }}`
      - `requiresHumanReview` ← `true`
      - `reviewStatus` ← `pending`

23. **Add storage for low-confidence review cases**
    - Add a **Data Table** node.
    - Name it **Store Low Confidence Cases**.
    - Use **Auto-map input data**.
    - Select or create a Data Table for review queue records.
    - Replace the placeholder table ID with your actual Data Table ID.
    - Connect **Prepare Low Confidence Data** → **Store Low Confidence Cases**.

24. **Add the low-confidence alert email**
    - Add an **Email Send** node.
    - Name it **Send Review Required Alert**.
    - Configure:
      - To: hiring manager email
      - From: system email
      - Subject indicating manual review required
      - HTML body containing candidate overview, concerns, detailed analysis, recommendation, and review warning
    - Connect **Store Low Confidence Cases** → **Send Review Required Alert**.

25. **Recommended correction for the low-confidence email**
    - As with the other email, replace `{{ $json.output.* }}` references with direct field references unless you intentionally wrap the payload in an `output` object.
    - If using `{{ $json.concerns.map(...) }}`, make sure `concerns` is always an array.

26. **Configure credentials**
    - **OpenAI**
      - Create one OpenAI credential in n8n
      - Reuse it for all five GPT-4o model nodes
    - **Email**
      - Configure SMTP, Gmail, or another supported mail transport in each Email Send node
    - **Data storage**
      - This workflow actually uses **n8n Data Tables**, not Google Sheets, despite the title/notes
      - Create/select two Data Tables:
        - candidate analysis results table
        - low-confidence review queue table

27. **Prepare Data Table schemas**
    - Suggested fields for analysis table:
      - candidateId, timestamp, overallScore, confidenceLevel, skillMatchScore, experienceScore, culturalFitScore, recommendation, detailedExplanation
    - Suggested fields for review queue:
      - candidateId, timestamp, overallScore, confidenceLevel, concerns, requiresHumanReview, reviewStatus

28. **Test the webhook**
    - Send a sample POST payload containing:
      - candidate identifier
      - resume text
      - job description
      - company culture or role requirements
    - Verify the orchestrator can infer enough information to call its tools correctly.

29. **Validate structured output**
    - Confirm the final result from the orchestrator contains every required schema field.
    - If parsing fails, refine the orchestrator system prompt and specialist prompts.

30. **Check routing behavior**
    - Test one payload expected to produce:
      - `confidenceLevel = high`
    - Test another expected to produce:
      - `confidenceLevel = low`
      - or score below/equal to 75
    - Verify each reaches the correct branch.

31. **Enable the workflow**
    - Once webhook, AI credentials, Data Table IDs, and email settings are working, activate the workflow.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows. All AI tools are internal tool nodes connected directly to the orchestrator.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API key (or compatible LLM) is required. | Prerequisite |
| Google Sheets are mentioned in the title and sticky notes, but the actual nodes used are n8n Data Tables, not Google Sheets nodes. | Important implementation mismatch |
| Email account credentials are required for both reporting nodes. | Prerequisite |
| Intended users: recruiters, HR operations teams, and talent acquisition leads. | Use case |
| Suggested customization: extend specialist agents with domain-specific scoring rubrics for technical or executive roles. | Customization |
| Benefit highlighted in notes: four specialist agents evaluate skills, experience, and cultural fit in parallel. | Design rationale |
| Setup guidance in notes: configure webhook URL, AI credentials, storage target IDs, email credentials, and confidence thresholds. | Operational setup |
| Confidence routing rationale: separate stronger matches from uncertain assessments while preserving low-confidence cases for human review. | Design rationale |

## Important Implementation Observations

1. **Storage mismatch with title/notes**
   - The workflow title mentions Google Sheets, but the implementation uses **Data Table** nodes.

2. **Email expression mismatch**
   - Both email nodes reference `{{$json.output.*}}`, but upstream Set/Data Table nodes appear to produce flat fields, not an `output` object.

3. **High-confidence branch drops strengths**
   - The high-confidence email tries to render `strengths`, but `Prepare Analysis Data` does not preserve that field.

4. **Validation tool input mismatch**
   - The code tool schema expects `resumeMetadata`, but the code reads from `query` directly, which may not match the actual payload shape passed by the agent.

5. **Webhook response timing**
   - Because the webhook waits for the last node, the caller may experience a long response time due to multi-agent LLM processing, storage, and email dispatch.