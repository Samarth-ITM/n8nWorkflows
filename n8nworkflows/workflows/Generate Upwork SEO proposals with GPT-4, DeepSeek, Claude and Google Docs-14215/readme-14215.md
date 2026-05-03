Generate Upwork SEO proposals with GPT-4, DeepSeek, Claude and Google Docs

https://n8nworkflows.xyz/workflows/generate-upwork-seo-proposals-with-gpt-4--deepseek--claude-and-google-docs-14215


# Generate Upwork SEO proposals with GPT-4, DeepSeek, Claude and Google Docs

# 1. Workflow Overview

This workflow generates tailored Upwork SEO proposals from a user-submitted job post. It collects job details through an n8n form, analyzes the post with GPT-4 Turbo, detects hidden screening questions, generates both screening answers and a personalized cover letter using DeepSeek plus Pinecone-backed context, reviews the draft against a proposal quality checklist, polishes the result with Claude 3.7 Sonnet, converts the final text into HTML, and saves both the final cover and Q&A into Google Docs.

Target use cases include:
- SEO freelancers responding to Upwork job posts
- Agencies standardizing proposal generation
- Operators who want AI-assisted drafting with quality control and document storage

The workflow is logically divided into these blocks:

## 1.1 Input Reception
Captures the Upwork job URL, raw job description, and job type from an n8n form trigger.

## 1.2 Job Analysis
Uses GPT-4 Turbo in an AI Agent node to extract structured job information, client context, and hidden screening questions.

## 1.3 Screening Question Detection and Answering
Uses DeepSeek plus Pinecone tools to answer hidden screening questions, formats the output as HTML, and appends it into a Google Doc.

## 1.4 Cover Letter Generation
Uses DeepSeek plus Pinecone tools to draft a personalized Upwork cover letter with keyword examples and one case study.

## 1.5 Cover Quality Review
Combines the generated cover with analyzed job details, then runs a DeepSeek quality review based on a 10-point proposal checklist.

## 1.6 Final Cover Polishing
Combines the original cover and QC feedback, then uses Claude 3.7 Sonnet to minimally revise the proposal while preserving tone and included examples.

## 1.7 Final Formatting and Storage
Converts the final polished HTML/markdown content and inserts it into a Google Doc.

## 1.8 Shared Retrieval Infrastructure
Provides two Pinecone vector-store tools, backed by OpenAI embeddings, for:
- case studies
- ranking keyword examples

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block is the user entry point. It exposes a form where the operator pastes an Upwork job URL, full job text, and selects a job type.

**Nodes Involved:**  
- Job Input Form

### Node Details

#### Job Input Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that starts the workflow when a web form is submitted.
- **Configuration choices:**  
  - Form title: **Upwork Job Details**
  - Description includes a Google Docs link labeled “Copy Cover from Google Doc”
  - Fields:
    - **Job URL**: text input
    - **Paste Job Details Here**: textarea
    - **Job Type**: dropdown with options:
      - SEO
      - Agency
      - Automation
- **Key expressions or variables used:**  
  Downstream nodes access submitted fields using keys such as:
  - `{{ $json['Job URL'] }}`
  - `{{ $json['Paste Job Details Here'] }}`
  - `{{ $json['Job Type'] }}`
- **Input and output connections:**  
  - No incoming connection; it is the workflow trigger
  - Outputs to **Job Details Analyzer**
- **Version-specific requirements:**  
  - Uses form trigger node version `2.2`
  - Requires n8n instance support for hosted forms/webhooks
- **Edge cases or potential failure types:**  
  - Missing or low-quality pasted job text reduces downstream extraction quality
  - Dropdown value is collected but not actually referenced later in expressions/prompts
  - Webhook availability depends on deployment and public URL configuration
- **Sub-workflow reference:**  
  None

---

## 2.2 Job Analysis

**Overview:**  
This block transforms the pasted raw job post into structured analysis. It extracts title, industry, SEO pain points, client history, budget, and explicit/hidden screening questions, while emphasizing any detection with warning markers.

**Nodes Involved:**  
- Job Details Analyzer
- GPT-4 Turbo LLM

### Node Details

#### Job Details Analyzer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent node that receives the raw job description and produces structured analysis.
- **Configuration choices:**  
  - Input text is taken from the form field:
    - `={{ $json['Paste Job Details Here'] }}`
  - Prompt is fully defined in the node
  - Output parser is enabled
  - Error handling is set to **continueRegularOutput**
  - The prompt instructs the model to extract:
    - Job Title
    - Industry Focus
    - Job Description
    - Primary SEO Problems
    - Current SEO Status
    - Skills & Expertise Required
    - Client Country
    - Client History
    - Industry Pattern Analysis
    - Budget Details
    - Preferred Experience
    - Deadlines/Timeline
    - Screening questions
    - Strategic Notes
  - It explicitly flags hidden screening questions with **⚠️ ATTENTION ⚠️**
- **Key expressions or variables used:**  
  - Input: `{{ $json['Paste Job Details Here'] }}`
  - Output is consumed downstream as `{{ $json.output }}`
- **Input and output connections:**  
  - Main input from **Job Input Form**
  - AI language model input from **GPT-4 Turbo LLM**
  - Main outputs to:
    - **Merge: Cover + Job Details**
    - **Screening Q&A Writer**
    - **Cover Letter Generator**
- **Version-specific requirements:**  
  - Uses AI Agent version `1.7`
  - Requires LangChain-compatible AI node support in n8n
- **Edge cases or potential failure types:**  
  - LLM may hallucinate structure when source text is incomplete
  - Output parser may fail if the model does not follow expected formatting
  - Large job descriptions may increase latency/cost
  - If the raw pasted text contains scraped UI clutter, extraction quality may degrade
  - Since error mode is continue-on-error, downstream nodes may still run on partial/empty outputs
- **Sub-workflow reference:**  
  None

#### GPT-4 Turbo LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model node providing the language model for the analyzer agent.
- **Configuration choices:**  
  - Model: **gpt-4-turbo**
  - Default options otherwise
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output connected to **Job Details Analyzer**
- **Version-specific requirements:**  
  - Uses type version `1.2`
  - Requires valid OpenAI credentials
- **Edge cases or potential failure types:**  
  - Invalid API key
  - OpenAI quota/rate limits
  - Model availability changes by account/region
- **Sub-workflow reference:**  
  None

---

## 2.3 Screening Question Detection and Answering

**Overview:**  
This block takes the structured job analysis, uses DeepSeek to answer any detected screening questions, enriches answers with Pinecone-retrieved examples if needed, and saves the result to Google Docs.

**Nodes Involved:**  
- Screening Q&A Writer
- DeepSeek LLM (Q&A Writer)
- Case Studies DB (Pinecone)
- Ranking Keywords DB (Pinecone)
- OpenAI Embeddings (Case Studies)
- OpenAI Embeddings (Keywords)
- Save Q&A to Docs

### Node Details

#### Screening Q&A Writer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent that reads analyzed job details and writes screening-question answers in HTML.
- **Configuration choices:**  
  - Input text:
    - `=Job Details: \n{{ $json.output }}`
  - Prompt instructs:
    - answer all identified screening questions directly
    - if none exist, explicitly say so
    - format output as HTML using `<p>` tags
    - structure as Question/Answer pairs
    - keep each answer under 200 words
    - optionally use examples from:
      - `websitewithrankingkeywords`
      - `casestudiesdatabase`
    - preserve Google Doc URLs from case studies exactly
- **Key expressions or variables used:**  
  - Consumes upstream analysis via `{{ $json.output }}`
  - Google Docs node later uses `{{ $json.data }}`
- **Input and output connections:**  
  - Main input from **Job Details Analyzer**
  - AI language model input from **DeepSeek LLM (Q&A Writer)**
  - AI tool inputs from:
    - **Case Studies DB (Pinecone)**
    - **Ranking Keywords DB (Pinecone)**
  - Main output to **Save Q&A to Docs**
- **Version-specific requirements:**  
  - Agent version `1.7`
  - Requires n8n AI tool connectivity for Pinecone tool use
- **Edge cases or potential failure types:**  
  - If no screening questions are extracted upstream, the output becomes a “none found” response
  - HTML may be malformed if the model ignores prompt constraints
  - Tool retrieval relevance depends on vector quality and embeddings
- **Sub-workflow reference:**  
  None

#### DeepSeek LLM (Q&A Writer)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek`  
  DeepSeek chat model for the screening-answer agent.
- **Configuration choices:**  
  - Default options only
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to **Screening Q&A Writer**
- **Version-specific requirements:**  
  - Requires DeepSeek credentials/API access
- **Edge cases or potential failure types:**  
  - Auth failure
  - API instability or timeout
  - Regional access restrictions depending on deployment
- **Sub-workflow reference:**  
  None

#### Case Studies DB (Pinecone)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Pinecone vector-store tool exposing case studies to AI agents.
- **Configuration choices:**  
  - Mode: **retrieve-as-tool**
  - Tool name: `casestudiesdatabase`
  - Index: `casestudiesdatabase`
  - Top K: `2`
  - Description emphasizes exact preservation of Google Doc URLs
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Embedding input from **OpenAI Embeddings (Case Studies)**
  - AI tool output to:
    - **Screening Q&A Writer**
    - **Cover Letter Generator**
- **Version-specific requirements:**  
  - Node version `1.2`
  - Pinecone index must already exist
- **Edge cases or potential failure types:**  
  - Missing index
  - Namespace/index mismatch
  - Retrieval may return low-quality or irrelevant case studies
- **Sub-workflow reference:**  
  None

#### Ranking Keywords DB (Pinecone)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Pinecone vector-store tool exposing ranking keyword examples to AI agents.
- **Configuration choices:**  
  - Mode: **retrieve-as-tool**
  - Tool name: `websitewithrankingkeywords`
  - Index: `websitewithrankingkeywords-v2`
  - Top K: `20`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Embedding input from **OpenAI Embeddings (Keywords)**
  - AI tool output to:
    - **Screening Q&A Writer**
    - **Cover Letter Generator**
- **Version-specific requirements:**  
  - Pinecone index must exist and be populated
- **Edge cases or potential failure types:**  
  - Wrong index selected
  - Top K may return too much noisy context
- **Sub-workflow reference:**  
  None

#### OpenAI Embeddings (Case Studies)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Embedding model node supplying vector embeddings to the case studies Pinecone node.
- **Configuration choices:**  
  - Default options
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI embedding output to **Case Studies DB (Pinecone)**
- **Version-specific requirements:**  
  - OpenAI credentials required
- **Edge cases or potential failure types:**  
  - Auth/rate limit errors
  - Embedding model availability changes
- **Sub-workflow reference:**  
  None

#### OpenAI Embeddings (Keywords)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Embedding model node supplying vector embeddings to the keyword Pinecone node.
- **Configuration choices:**  
  - Default options
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI embedding output to **Ranking Keywords DB (Pinecone)**
- **Version-specific requirements:**  
  - OpenAI credentials required
- **Edge cases or potential failure types:**  
  Same as above
- **Sub-workflow reference:**  
  None

#### Save Q&A to Docs
- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Updates a Google Doc by inserting generated screening Q&A content.
- **Configuration choices:**  
  - Operation: **update**
  - Action: **insert**
  - Target document URL placeholder:
    - `YOUR_GOOGLE_DOC_URL_FOR_QA`
  - Inserted text includes:
    - a separator line
    - the Job URL from the form
    - generated Q&A content from `{{ $json.data }}`
- **Key expressions or variables used:**  
  - `{{ $('Job Input Form').item.json['Job URL'] }}`
  - `{{ $json.data }}`
- **Input and output connections:**  
  - Main input from **Screening Q&A Writer**
- **Version-specific requirements:**  
  - Requires Google Docs credentials
- **Edge cases or potential failure types:**  
  - Placeholder URL not replaced
  - OAuth credentials missing/expired
  - Document permission denied
  - If the agent returns `output` instead of `data` depending on node behavior/version, the expression may insert empty content
- **Sub-workflow reference:**  
  None

---

## 2.4 Cover Letter Generation

**Overview:**  
This block drafts the actual proposal cover letter using the analyzed job details and retrieved examples from Pinecone. It is optimized for Upwork response performance and constrained to preserve exact case study URLs and formatting.

**Nodes Involved:**  
- Cover Letter Generator
- DeepSeek LLM (Cover Writer)
- Case Studies DB (Pinecone)
- Ranking Keywords DB (Pinecone)

### Node Details

#### Cover Letter Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent that drafts a personalized Upwork cover letter.
- **Configuration choices:**  
  - Input text:
    - `={{ $json.output }}`
  - Prompt instructs the model to:
    - write a 150–250 word personalized proposal
    - mirror the client’s problem
    - use exact job terminology
    - avoid placeholders
    - ask a project-specific question
    - end with a CTA
    - avoid portfolio links except exact retrieved examples
    - include:
      - **2 examples**
      - **1 case study**
    - preserve case study URLs exactly
    - never wrap URLs in angle brackets
    - keep formatting of inserted examples exactly
- **Key expressions or variables used:**  
  - Input from analyzer via `{{ $json.output }}`
- **Input and output connections:**  
  - Main input from **Job Details Analyzer**
  - AI language model from **DeepSeek LLM (Cover Writer)**
  - AI tools from:
    - **Case Studies DB (Pinecone)**
    - **Ranking Keywords DB (Pinecone)**
  - Main outputs to:
    - **Merge: QC Feedback + Cover**
    - **Merge: Cover + Job Details**
- **Version-specific requirements:**  
  - Agent version `1.7`
- **Edge cases or potential failure types:**  
  - Model may exceed or ignore word count
  - Retrieved examples may not match the client industry well
  - Prompt contains conflicting instructions: early section says no portfolio links, later section requires examples/case study; result quality depends on model interpretation
  - URL formatting may still break if model reformats retrieved content
- **Sub-workflow reference:**  
  None

#### DeepSeek LLM (Cover Writer)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek`
- **Configuration choices:**  
  - Default options
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to **Cover Letter Generator**
- **Version-specific requirements:**  
  Requires DeepSeek credentials
- **Edge cases or potential failure types:**  
  Same DeepSeek API issues as above
- **Sub-workflow reference:**  
  None

---

## 2.5 Cover Quality Review

**Overview:**  
This block packages the generated cover together with the analyzed job details, then runs a structured quality review. The goal is not to rewrite the cover yet, but to produce targeted improvement instructions.

**Nodes Involved:**  
- Merge: Cover + Job Details
- Prepare for QC Review
- Cover Quality Checker
- DeepSeek LLM (QC Checker)

### Node Details

#### Merge: Cover + Job Details
- **Type and technical role:** `n8n-nodes-base.merge`  
  Merges the analyzed job details and generated cover into a single stream.
- **Configuration choices:**  
  - Default merge parameters are used
- **Key expressions or variables used:**  
  None directly in the node configuration
- **Input and output connections:**  
  - Input 0 from **Job Details Analyzer**
  - Input 1 from **Cover Letter Generator**
  - Output to **Prepare for QC Review**
- **Version-specific requirements:**  
  - Merge node version `3`
  - Behavior depends on n8n default merge mode for this version
- **Edge cases or potential failure types:**  
  - If one branch outputs zero items, merge behavior may produce no data or mismatched pairing
  - Default mode ambiguity can matter when scaling beyond one item
- **Sub-workflow reference:**  
  None

#### Prepare for QC Review
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Aggregates all item data into a single structure for the QC reviewer.
- **Configuration choices:**  
  - Aggregate mode: **aggregateAllItemData**
- **Key expressions or variables used:**  
  Produces combined payload later referenced as `{{ $json.data }}`
- **Input and output connections:**  
  - Input from **Merge: Cover + Job Details**
  - Output to **Cover Quality Checker**
- **Version-specific requirements:**  
  - Aggregate node version `1`
- **Edge cases or potential failure types:**  
  - If merged data is malformed, the aggregated blob may be hard for the LLM to interpret
- **Sub-workflow reference:**  
  None

#### Cover Quality Checker
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent that reviews the generated cover against a 10-point checklist.
- **Configuration choices:**  
  - Input:
    - `={{ $json.data }}`
  - Error handling: **continueRegularOutput**
  - Prompt checks:
    1. client name or greeting
    2. terminology alignment
    3. strong opening
    4. job-description reference
    5. keyword utilization
    6. industry relevance
    7. task-specific experience
    8. skill match
    9. process outline
    10. CTA
  - Output format requested:
    - “Required Changes:” list
    - or “No changes required...”
- **Key expressions or variables used:**  
  - `{{ $json.data }}`
- **Input and output connections:**  
  - Main input from **Prepare for QC Review**
  - AI language model from **DeepSeek LLM (QC Checker)**
  - Output to **Merge: QC Feedback + Cover**
- **Version-specific requirements:**  
  - Agent version `1.7`
- **Edge cases or potential failure types:**  
  - If agent output is empty due to error and continue-on-error is enabled, downstream polish may be based on incomplete feedback
  - QC may focus on stylistic details instead of conversion-critical issues depending on model behavior
- **Sub-workflow reference:**  
  None

#### DeepSeek LLM (QC Checker)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek`
- **Configuration choices:**  
  - Default options
- **Input and output connections:**  
  - AI language model output to **Cover Quality Checker**
- **Version-specific requirements:**  
  Requires DeepSeek credentials
- **Edge cases or potential failure types:**  
  Standard DeepSeek auth/rate/timeout issues
- **Sub-workflow reference:**  
  None

---

## 2.6 Final Cover Polishing

**Overview:**  
This block combines the QC feedback with the original cover and asks Claude 3.7 Sonnet to apply minimal revisions. It is explicitly instructed to preserve examples and remove angle brackets from URLs.

**Nodes Involved:**  
- Merge: QC Feedback + Cover
- Prepare for Final Polish
- Final Cover Polish
- Claude 3.7 Sonnet LLM

### Node Details

#### Merge: QC Feedback + Cover
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the QC feedback and original cover before final polishing.
- **Configuration choices:**  
  - Default merge behavior
- **Key expressions or variables used:**  
  None directly
- **Input and output connections:**  
  - Input 0 from **Cover Quality Checker**
  - Input 1 from **Cover Letter Generator**
  - Output to **Prepare for Final Polish**
- **Version-specific requirements:**  
  - Merge node version `3`
- **Edge cases or potential failure types:**  
  - If QC output is missing, final polish may use only original cover or fail to produce meaningful changes
- **Sub-workflow reference:**  
  None

#### Prepare for Final Polish
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Aggregates merged QC + cover data into a single payload for Claude.
- **Configuration choices:**  
  - Aggregate mode: **aggregateAllItemData**
- **Key expressions or variables used:**  
  Produces payload consumed as `{{ $json.data }}`
- **Input and output connections:**  
  - Input from **Merge: QC Feedback + Cover**
  - Output to **Final Cover Polish**
- **Version-specific requirements:**  
  - Aggregate version `1`
- **Edge cases or potential failure types:**  
  - Same general aggregate data-shape issues as earlier
- **Sub-workflow reference:**  
  None

#### Final Cover Polish
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent that revises the original cover using QC suggestions while preserving tone and examples.
- **Configuration choices:**  
  - Input:
    - `={{ $json.data }}`
  - Error handling: **continueRegularOutput**
  - Prompt instructs:
    - make minimal changes
    - keep portfolio URLs/examples
    - remove angle brackets around URLs
    - do not use `<a>` tags
    - return HTML using `<p>`, optionally `<ul>/<li>`
- **Key expressions or variables used:**  
  - `{{ $json.data }}`
  - Output consumed downstream as `{{ $json.output }}`
- **Input and output connections:**  
  - Main input from **Prepare for Final Polish**
  - AI language model from **Claude 3.7 Sonnet LLM**
  - Main output to **HTML Converter (Final Cover)**
- **Version-specific requirements:**  
  - Agent version `1.7`
  - Depends on Anthropic model support in n8n
- **Edge cases or potential failure types:**  
  - Prompt asks for HTML but next node is named markdown converter; output/rendering assumptions may be inconsistent
  - Claude may still normalize formatting in unexpected ways
  - Continue-on-error may pass unusable content to downstream formatting
- **Sub-workflow reference:**  
  None

#### Claude 3.7 Sonnet LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Anthropic chat model used for final proposal polishing.
- **Configuration choices:**  
  - Model: `claude-3-7-sonnet-20250219`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to **Final Cover Polish**
- **Version-specific requirements:**  
  - Anthropic credentials required
  - Model availability may vary by account
- **Edge cases or potential failure types:**  
  - Auth errors
  - Quota limits
  - Model deprecation/name changes over time
- **Sub-workflow reference:**  
  None

---

## 2.7 Final Formatting and Storage

**Overview:**  
This block transforms the polished output and appends it to a Google Doc for final use. It is the last delivery stage for the cover letter.

**Nodes Involved:**  
- HTML Converter (Final Cover)
- Save Final Cover to Docs

### Node Details

#### HTML Converter (Final Cover)
- **Type and technical role:** `n8n-nodes-base.markdown`  
  Content conversion node. Despite its name, it is configured using the `html` parameter and receives the polished content from the prior agent.
- **Configuration choices:**  
  - HTML input:
    - `={{ $json.output }}`
  - Default options
- **Key expressions or variables used:**  
  - `{{ $json.output }}`
- **Input and output connections:**  
  - Main input from **Final Cover Polish**
  - Main output to **Save Final Cover to Docs**
- **Version-specific requirements:**  
  - Markdown node version `1`
- **Edge cases or potential failure types:**  
  - If upstream already returns HTML, conversion may be redundant or may transform content unexpectedly
  - Output field naming must match what Google Docs insert action expects downstream
- **Sub-workflow reference:**  
  None

#### Save Final Cover to Docs
- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Appends the final cover letter to a Google Doc.
- **Configuration choices:**  
  - Operation: **update**
  - Authentication: **oAuth2**
  - Document URL placeholder:
    - `YOUR_GOOGLE_DOC_URL_FOR_COVERS`
  - Insert text:
    - `{{ $json.data }}`
- **Key expressions or variables used:**  
  - `{{ $json.data }}`
- **Input and output connections:**  
  - Input from **HTML Converter (Final Cover)**
- **Version-specific requirements:**  
  - Requires Google Docs OAuth2 credentials
- **Edge cases or potential failure types:**  
  - Placeholder URL not replaced
  - OAuth scope/permission issues
  - If the markdown node outputs a different property than `data`, inserted content may be blank
- **Sub-workflow reference:**  
  None

---

## 2.8 Shared Retrieval Infrastructure

**Overview:**  
These nodes support both the screening-answer and cover-letter branches. They retrieve semantically relevant examples from Pinecone using OpenAI embeddings.

**Nodes Involved:**  
- Case Studies DB (Pinecone)
- OpenAI Embeddings (Case Studies)
- Ranking Keywords DB (Pinecone)
- OpenAI Embeddings (Keywords)

These nodes were already detailed above because they are shared across multiple blocks.

---

## 2.9 Documentation and Visual Notes

**Overview:**  
These sticky notes do not affect execution but are important for maintainability. They describe workflow purpose, setup steps, and block-level meaning.

**Nodes Involved:**  
- Main Description Note
- Note: Job Input Form
- Note: Analyze Job Post
- Note: Detect and Answer Screening
- Note: Generate Cover Letter
- Note: Quality Check
- Note: Polish and Improve
- Note: Format and Save

### Node Details

#### Main Description Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Non-executable documentation node summarizing workflow purpose, setup, and customization.
- **Configuration choices:**  
  Contains:
  - workflow description
  - setup checklist
  - customization suggestions
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Note: Job Input Form
- Sticky note documenting Block 1.

#### Note: Analyze Job Post
- Sticky note documenting Block 2.

#### Note: Detect and Answer Screening
- Sticky note documenting Block 3.

#### Note: Generate Cover Letter
- Sticky note documenting Block 4.

#### Note: Quality Check
- Sticky note documenting Block 5.

#### Note: Polish and Improve
- Sticky note documenting Block 6.

#### Note: Format and Save
- Sticky note documenting Block 7.

All sticky notes are non-executable and have no runtime behavior.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Description Note | n8n-nodes-base.stickyNote | Visual documentation and setup guidance |  |  | ## 📄 AI-Powered Upwork Proposal Generator<br><br>This workflow automatically generates high-converting Upwork proposals. User submits job details via a form, AI analyzes the job, detects hidden screening questions, writes a cover letter with real case studies and ranking keywords, runs a quality check, improves it, and saves everything to Google Docs.<br><br>## ⚙️ How it works<br><br>1. User submits job URL, job details, and job type via form<br>2. GPT-4 Turbo analyzes the job post and extracts key info<br>3. DeepSeek detects hidden screening questions and writes answers<br>4. DeepSeek generates a personalized cover letter using Pinecone case studies and ranking keywords database<br>5. Cover letter and job details are merged for quality review<br>6. DeepSeek reviews cover against a 10-point quality checklist<br>7. QC feedback is merged with original cover letter<br>8. Claude 3.7 Sonnet polishes the cover with minimal changes<br>9. Both cover letter and Q&A are converted to HTML<br>10. Final cover and Q&A are saved to separate Google Docs.<br><br>## 🛠️ Setup steps<br><br>☐ Add OpenAI API key (GPT-4 Turbo + Embeddings nodes)<br>☐ Add Anthropic API key (Claude 3.7 Sonnet node)<br>☐ Add DeepSeek API key (3 DeepSeek LLM nodes)<br>☐ Connect Google Docs OAuth credentials<br>☐ Replace YOUR_GOOGLE_DOC_URL_FOR_COVERS in Save Cover node<br>☐ Replace YOUR_GOOGLE_DOC_URL_FOR_QA in Save Q&A node<br>☐ Connect Pinecone and set up casestudiesdatabase index<br>☐ Connect Pinecone and set up websitewithrankingkeywords-v2 index<br>☐ Add your case studies and ranking keywords to Pinecone<br><br>## ✏️ Customize<br><br>- Job types: add/edit dropdown options in Job Input Form<br>- AI model: swap DeepSeek with GPT-4o or Claude in any agent<br>- QC checklist: edit system prompt in Cover Quality Checker<br>- Examples format: change portfolio format in Cover Letter Generator<br>- Top results: adjust topK in Pinecone nodes for more/less context |
| Note: Job Input Form | n8n-nodes-base.stickyNote | Visual note for input block |  |  | ## 1. Job Input Form<br><br>User submits job URL, raw job description, and job type (SEO / Agency / Automation). |
| Note: Analyze Job Post | n8n-nodes-base.stickyNote | Visual note for analysis block |  |  | ## 2. Analyze Job Post<br><br>GPT-4 Turbo extracts job title, industry, budget, client history, required skills, and flags any hidden screening questions with ⚠️ markers. |
| Note: Detect and Answer Screening | n8n-nodes-base.stickyNote | Visual note for screening block |  |  | ## 3. Detect & Answer Screening Questions<br><br>DeepSeek reads hidden questions from job analysis and writes direct answers using case studies DB. Converts to HTML and saves to Google Docs. |
| Note: Generate Cover Letter | n8n-nodes-base.stickyNote | Visual note for cover generation block |  |  | ## 4. Generate Cover Letter<br><br>DeepSeek writes a personalized 150-250 word cover letter. Pulls 2 keyword examples and 1 case study from Pinecone based on client's industry. |
| Note: Quality Check | n8n-nodes-base.stickyNote | Visual note for QC block |  |  | ## 5. Quality Check<br><br>Merges cover with job details and runs a 10-point quality checklist review via DeepSeek. Returns list of specific improvements needed. |
| Note: Polish and Improve | n8n-nodes-base.stickyNote | Visual note for final polishing block |  |  | ## 6. Polish & Improve Cover<br><br>Merges QC feedback with original cover letter. Claude 3.7 Sonnet applies improvements with minimal changes to preserve client's tone and language. |
| Note: Format and Save | n8n-nodes-base.stickyNote | Visual note for save block |  |  | ## 7. Format & Save Final Cover<br><br>Converts polished cover to clean HTML format and saves directly to Google Docs — ready to copy. |
| Job Input Form | n8n-nodes-base.formTrigger | Workflow entry form for job data |  | Job Details Analyzer | ## 1. Job Input Form<br><br>User submits job URL, raw job description, and job type (SEO / Agency / Automation). |
| Job Details Analyzer | @n8n/n8n-nodes-langchain.agent | Extracts structured job insights and screening questions | Job Input Form; GPT-4 Turbo LLM | Merge: Cover + Job Details; Screening Q&A Writer; Cover Letter Generator | ## 2. Analyze Job Post<br><br>GPT-4 Turbo extracts job title, industry, budget, client history, required skills, and flags any hidden screening questions with ⚠️ markers. |
| GPT-4 Turbo LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for job analysis |  | Job Details Analyzer | ## 2. Analyze Job Post<br><br>GPT-4 Turbo extracts job title, industry, budget, client history, required skills, and flags any hidden screening questions with ⚠️ markers. |
| Screening Q&A Writer | @n8n/n8n-nodes-langchain.agent | Generates HTML answers to screening questions | Job Details Analyzer; DeepSeek LLM (Q&A Writer); Case Studies DB (Pinecone); Ranking Keywords DB (Pinecone) | Save Q&A to Docs | ## 3. Detect & Answer Screening Questions<br><br>DeepSeek reads hidden questions from job analysis and writes direct answers using case studies DB. Converts to HTML and saves to Google Docs. |
| DeepSeek LLM (Q&A Writer) | @n8n/n8n-nodes-langchain.lmChatDeepSeek | LLM backend for screening Q&A generation |  | Screening Q&A Writer | ## 3. Detect & Answer Screening Questions<br><br>DeepSeek reads hidden questions from job analysis and writes direct answers using case studies DB. Converts to HTML and saves to Google Docs. |
| Save Q&A to Docs | n8n-nodes-base.googleDocs | Appends screening Q&A to Google Docs | Screening Q&A Writer |  | ## 3. Detect & Answer Screening Questions<br><br>DeepSeek reads hidden questions from job analysis and writes direct answers using case studies DB. Converts to HTML and saves to Google Docs. |
| Cover Letter Generator | @n8n/n8n-nodes-langchain.agent | Drafts the Upwork cover letter with retrieved examples | Job Details Analyzer; DeepSeek LLM (Cover Writer); Case Studies DB (Pinecone); Ranking Keywords DB (Pinecone) | Merge: QC Feedback + Cover; Merge: Cover + Job Details | ## 4. Generate Cover Letter<br><br>DeepSeek writes a personalized 150-250 word cover letter. Pulls 2 keyword examples and 1 case study from Pinecone based on client's industry. |
| DeepSeek LLM (Cover Writer) | @n8n/n8n-nodes-langchain.lmChatDeepSeek | LLM backend for cover generation |  | Cover Letter Generator | ## 4. Generate Cover Letter<br><br>DeepSeek writes a personalized 150-250 word cover letter. Pulls 2 keyword examples and 1 case study from Pinecone based on client's industry. |
| Merge: Cover + Job Details | n8n-nodes-base.merge | Combines analyzed job details with generated cover | Job Details Analyzer; Cover Letter Generator | Prepare for QC Review | ## 5. Quality Check<br><br>Merges cover with job details and runs a 10-point quality checklist review via DeepSeek. Returns list of specific improvements needed. |
| Prepare for QC Review | n8n-nodes-base.aggregate | Aggregates merged inputs for QC review | Merge: Cover + Job Details | Cover Quality Checker | ## 5. Quality Check<br><br>Merges cover with job details and runs a 10-point quality checklist review via DeepSeek. Returns list of specific improvements needed. |
| Cover Quality Checker | @n8n/n8n-nodes-langchain.agent | Reviews cover against a 10-point proposal checklist | Prepare for QC Review; DeepSeek LLM (QC Checker) | Merge: QC Feedback + Cover | ## 5. Quality Check<br><br>Merges cover with job details and runs a 10-point quality checklist review via DeepSeek. Returns list of specific improvements needed. |
| DeepSeek LLM (QC Checker) | @n8n/n8n-nodes-langchain.lmChatDeepSeek | LLM backend for proposal quality review |  | Cover Quality Checker | ## 5. Quality Check<br><br>Merges cover with job details and runs a 10-point quality checklist review via DeepSeek. Returns list of specific improvements needed. |
| Merge: QC Feedback + Cover | n8n-nodes-base.merge | Combines QC feedback with original cover | Cover Quality Checker; Cover Letter Generator | Prepare for Final Polish | ## 6. Polish & Improve Cover<br><br>Merges QC feedback with original cover letter. Claude 3.7 Sonnet applies improvements with minimal changes to preserve client's tone and language. |
| Prepare for Final Polish | n8n-nodes-base.aggregate | Aggregates cover and QC data for final revision | Merge: QC Feedback + Cover | Final Cover Polish | ## 6. Polish & Improve Cover<br><br>Merges QC feedback with original cover letter. Claude 3.7 Sonnet applies improvements with minimal changes to preserve client's tone and language. |
| Final Cover Polish | @n8n/n8n-nodes-langchain.agent | Applies minimal improvements and outputs HTML | Prepare for Final Polish; Claude 3.7 Sonnet LLM | HTML Converter (Final Cover) | ## 6. Polish & Improve Cover<br><br>Merges QC feedback with original cover letter. Claude 3.7 Sonnet applies improvements with minimal changes to preserve client's tone and language. |
| Claude 3.7 Sonnet LLM | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for final cover polishing |  | Final Cover Polish | ## 6. Polish & Improve Cover<br><br>Merges QC feedback with original cover letter. Claude 3.7 Sonnet applies improvements with minimal changes to preserve client's tone and language. |
| HTML Converter (Final Cover) | n8n-nodes-base.markdown | Converts final polished content for storage | Final Cover Polish | Save Final Cover to Docs | ## 7. Format & Save Final Cover<br><br>Converts polished cover to clean HTML format and saves directly to Google Docs — ready to copy. |
| Save Final Cover to Docs | n8n-nodes-base.googleDocs | Appends final cover to Google Docs | HTML Converter (Final Cover) |  | ## 7. Format & Save Final Cover<br><br>Converts polished cover to clean HTML format and saves directly to Google Docs — ready to copy. |
| Case Studies DB (Pinecone) | @n8n/n8n-nodes-langchain.vectorStorePinecone | AI retrieval tool for case study examples | OpenAI Embeddings (Case Studies) | Screening Q&A Writer; Cover Letter Generator |  |
| OpenAI Embeddings (Case Studies) | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model for case study retrieval |  | Case Studies DB (Pinecone) |  |
| Ranking Keywords DB (Pinecone) | @n8n/n8n-nodes-langchain.vectorStorePinecone | AI retrieval tool for keyword-ranking examples | OpenAI Embeddings (Keywords) | Screening Q&A Writer; Cover Letter Generator |  |
| OpenAI Embeddings (Keywords) | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model for keyword retrieval |  | Ranking Keywords DB (Pinecone) |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node named `Job Input Form`.**
   - Type: `Form Trigger`
   - Set form title to **Upwork Job Details**
   - Add form description with this link text:
     - `Copy Cover from Google Doc`
   - Add fields:
     1. Text field: `Job URL`
     2. Textarea: `Paste Job Details Here`
     3. Dropdown: `Job Type`
        - SEO
        - Agency
        - Automation

3. **Add an OpenAI Chat Model node named `GPT-4 Turbo LLM`.**
   - Type: OpenAI Chat Model
   - Select model: `gpt-4-turbo`
   - Attach valid OpenAI credentials

4. **Add an AI Agent node named `Job Details Analyzer`.**
   - Type: LangChain Agent
   - Set text input to:
     - `{{ $json['Paste Job Details Here'] }}`
   - Set prompt type to defined/system prompt
   - Paste the extraction prompt so it requests:
     - structured job analysis
     - industry focus
     - client history
     - industry pattern analysis
     - screening question detection
     - strategic notes
     - explicit warning markers for hidden questions
   - Enable output parser
   - Set error handling to continue regular output

5. **Connect `Job Input Form` → `Job Details Analyzer`.**

6. **Connect `GPT-4 Turbo LLM` to `Job Details Analyzer` as the AI language model.**

7. **Add a DeepSeek Chat Model node named `DeepSeek LLM (Q&A Writer)`.**
   - Configure with valid DeepSeek credentials

8. **Add a DeepSeek Chat Model node named `DeepSeek LLM (Cover Writer)`.**
   - Configure with valid DeepSeek credentials

9. **Add a DeepSeek Chat Model node named `DeepSeek LLM (QC Checker)`.**
   - Configure with valid DeepSeek credentials

10. **Add an Anthropic Chat Model node named `Claude 3.7 Sonnet LLM`.**
    - Select model `claude-3-7-sonnet-20250219`
    - Attach Anthropic credentials

11. **Add an OpenAI Embeddings node named `OpenAI Embeddings (Case Studies)`.**
    - Use valid OpenAI credentials

12. **Add an OpenAI Embeddings node named `OpenAI Embeddings (Keywords)`.**
    - Use valid OpenAI credentials

13. **Add a Pinecone Vector Store node named `Case Studies DB (Pinecone)`.**
    - Mode: `retrieve-as-tool`
    - Tool name: `casestudiesdatabase`
    - Index: `casestudiesdatabase`
    - Top K: `2`
    - Tool description should mention that Google Doc URLs must remain unchanged

14. **Connect `OpenAI Embeddings (Case Studies)` → `Case Studies DB (Pinecone)` as AI embedding input.**

15. **Add a Pinecone Vector Store node named `Ranking Keywords DB (Pinecone)`.**
    - Mode: `retrieve-as-tool`
    - Tool name: `websitewithrankingkeywords`
    - Index: `websitewithrankingkeywords-v2`
    - Top K: `20`

16. **Connect `OpenAI Embeddings (Keywords)` → `Ranking Keywords DB (Pinecone)` as AI embedding input.**

17. **Add an AI Agent node named `Screening Q&A Writer`.**
    - Input text:
      - `Job Details: {{ $json.output }}`
    - System instructions should:
      - answer screening questions directly
      - say no screening questions exist if none are found
      - format output in HTML with `<p>` tags
      - use Question then Answer structure
      - keep answers under 200 words
      - allow use of examples from both Pinecone tools
      - preserve case study Google Docs URLs exactly

18. **Connect `Job Details Analyzer` → `Screening Q&A Writer`.**

19. **Connect `DeepSeek LLM (Q&A Writer)` to `Screening Q&A Writer` as AI language model.**

20. **Connect `Case Studies DB (Pinecone)` to `Screening Q&A Writer` as AI tool.**

21. **Connect `Ranking Keywords DB (Pinecone)` to `Screening Q&A Writer` as AI tool.**

22. **Add a Google Docs node named `Save Q&A to Docs`.**
    - Operation: `Update`
    - Action: `Insert`
    - Set document URL to your real Q&A Google Doc
    - Insert text like:
      - separator line
      - submitted job URL
      - generated Q&A content
    - Example expressions:
      - `{{ $('Job Input Form').item.json['Job URL'] }}`
      - `{{ $json.data }}`

23. **Connect `Screening Q&A Writer` → `Save Q&A to Docs`.**

24. **Add an AI Agent node named `Cover Letter Generator`.**
    - Input text:
      - `{{ $json.output }}`
    - Add a detailed prompt that instructs the model to:
      - write a 150–250 word Upwork cover letter
      - personalize the opening
      - mirror client pain points
      - ask a project-specific question
      - end with a CTA
      - use exact job terminology
      - maintain tone close to the source post
      - retrieve and insert 2 ranking examples and 1 case study
      - preserve exact URLs
      - never wrap URLs with `< >`
      - avoid placeholder URLs

25. **Connect `Job Details Analyzer` → `Cover Letter Generator`.**

26. **Connect `DeepSeek LLM (Cover Writer)` to `Cover Letter Generator` as AI language model.**

27. **Connect `Case Studies DB (Pinecone)` to `Cover Letter Generator` as AI tool.**

28. **Connect `Ranking Keywords DB (Pinecone)` to `Cover Letter Generator` as AI tool.**

29. **Add a Merge node named `Merge: Cover + Job Details`.**
    - Keep default merge settings unless you want a stricter merge strategy
    - Connect:
      - `Job Details Analyzer` → Input 1
      - `Cover Letter Generator` → Input 2

30. **Add an Aggregate node named `Prepare for QC Review`.**
    - Aggregate mode: `Aggregate all item data`

31. **Connect `Merge: Cover + Job Details` → `Prepare for QC Review`.**

32. **Add an AI Agent node named `Cover Quality Checker`.**
    - Input text:
      - `{{ $json.data }}`
    - Set error handling to continue regular output
    - Prompt should evaluate the cover against the 10-point checklist:
      - salutation/client name
      - terminology alignment
      - strong opening
      - direct reference to job needs
      - keyword usage
      - industry relevance
      - task-specific experience
      - skills match
      - process/solution outline
      - CTA
    - Ask it to return only actionable “Required Changes” bullets, or say no changes are required

33. **Connect `Prepare for QC Review` → `Cover Quality Checker`.**

34. **Connect `DeepSeek LLM (QC Checker)` to `Cover Quality Checker` as AI language model.**

35. **Add a Merge node named `Merge: QC Feedback + Cover`.**
    - Connect:
      - `Cover Quality Checker` → Input 1
      - `Cover Letter Generator` → Input 2

36. **Add an Aggregate node named `Prepare for Final Polish`.**
    - Aggregate mode: `Aggregate all item data`

37. **Connect `Merge: QC Feedback + Cover` → `Prepare for Final Polish`.**

38. **Add an AI Agent node named `Final Cover Polish`.**
    - Input text:
      - `{{ $json.data }}`
    - Set error handling to continue regular output
    - Prompt should instruct the model to:
      - apply QC suggestions with minimal changes
      - preserve examples and URLs
      - remove angle brackets from URLs
      - avoid `<a>` tags
      - format output as HTML using `<p>` tags and optionally `<ul>/<li>`

39. **Connect `Prepare for Final Polish` → `Final Cover Polish`.**

40. **Connect `Claude 3.7 Sonnet LLM` to `Final Cover Polish` as AI language model.**

41. **Add a Markdown/HTML conversion node named `HTML Converter (Final Cover)`.**
    - Configure its HTML input from:
      - `{{ $json.output }}`

42. **Connect `Final Cover Polish` → `HTML Converter (Final Cover)`.**

43. **Add a Google Docs node named `Save Final Cover to Docs`.**
    - Operation: `Update`
    - Authentication: `OAuth2`
    - Set document URL to your actual Google Doc for covers
    - Insert text from the converter output, such as:
      - `{{ $json.data }}`

44. **Connect `HTML Converter (Final Cover)` → `Save Final Cover to Docs`.**

45. **Add sticky notes for documentation if desired.**
    - Create notes for:
      - main workflow description
      - job input form
      - analyze job post
      - screening
      - cover generation
      - quality check
      - polish and improve
      - format and save

46. **Configure credentials.**
    - OpenAI:
      - one credential usable by GPT-4 Turbo and both embeddings nodes
    - DeepSeek:
      - one credential reusable across all three DeepSeek chat nodes
    - Anthropic:
      - credential for Claude 3.7 Sonnet
    - Google Docs OAuth2:
      - must allow document update access
    - Pinecone:
      - must reference the exact indexes:
        - `casestudiesdatabase`
        - `websitewithrankingkeywords-v2`

47. **Populate Pinecone before testing.**
    - `casestudiesdatabase` should contain:
      - industry/client case studies
      - exact Google Doc URLs if referenced
    - `websitewithrankingkeywords-v2` should contain:
      - website URLs
      - ranking keywords
      - content relevant to industry matching

48. **Replace placeholders before production use.**
    - Replace:
      - `YOUR_GOOGLE_DOC_URL_FOR_QA`
      - `YOUR_GOOGLE_DOC_URL_FOR_COVERS`

49. **Run a test submission through the form.**
    - Confirm:
      - analyzer outputs structured fields
      - screening Q&A is inserted into the Q&A doc
      - cover letter is generated
      - QC feedback appears
      - final polished cover is inserted into the final Google Doc

50. **Recommended hardening after rebuild.**
    - Validate merge modes explicitly
    - Add empty-output checks before Google Docs nodes
    - Add fallback handling when no screening questions exist
    - Consider referencing `Job Type` in prompts if it should influence output
    - Verify whether downstream Google Docs expressions should use `output`, `data`, or another property based on actual runtime payloads

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Copy Cover from Google Doc | https://docs.google.com/document/d/12O0BoOkfgYhKpDHu0B_5p0lgfzFxvGL-Jqwm3skTnz4/edit?tab=t.0 |
| Setup requires OpenAI API key for GPT-4 Turbo and embeddings nodes | Workflow setup note |
| Setup requires Anthropic API key for Claude 3.7 Sonnet | Workflow setup note |
| Setup requires DeepSeek API key for the 3 DeepSeek LLM nodes | Workflow setup note |
| Replace the placeholder Google Doc URLs in both Google Docs nodes before use | Workflow setup note |
| Pinecone indexes expected: `casestudiesdatabase` and `websitewithrankingkeywords-v2` | Workflow setup note |
| Suggested customizations include editing job types, swapping AI models, changing QC checklist, adjusting examples format, and tuning Pinecone `topK` | Workflow customization note |

## Additional implementation notes
- The workflow has **one explicit entry point**: `Job Input Form`.
- There are **no sub-workflow nodes** and no workflow-invoking nodes.
- The `Job Type` field is collected but is not currently used in downstream prompts or logic.
- Both merge nodes rely on default merge behavior; if you adapt this workflow for batches or multiple simultaneous items, set merge mode explicitly.
- The Google Docs nodes use placeholder URLs and should be considered incomplete until replaced.
- There is a possible field-name mismatch risk in Google Docs insert expressions depending on actual output structure from the upstream AI/conversion nodes. Validate with execution data before production rollout.