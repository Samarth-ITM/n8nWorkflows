Evaluate job fit and generate application assets from Telegram links with OpenAI, Pinecone, Apify and Google Sheets

https://n8nworkflows.xyz/workflows/evaluate-job-fit-and-generate-application-assets-from-telegram-links-with-openai--pinecone--apify-and-google-sheets-14667


# Evaluate job fit and generate application assets from Telegram links with OpenAI, Pinecone, Apify and Google Sheets

Now I have a thorough understanding of the workflow. Let me write the comprehensive reference document. 1. Workflow Overview

This workflow automates the end-to-end job application process for a candidate. It is triggered when a user sends a job posting link via Telegram, and it proceeds through URL validation, web scraping, AI-driven job-fit evaluation using resume data stored in a Pinecone vector database, human-in-the-loop approval, and automatic generation of application materials (cover letter, HR email draft, Google Sheets tracking entry). The workflow uses OpenAI (GPT-5-mini) for all AI reasoning, Apify for web scraping, Pinecone for vector-based resume retrieval, and Google Workspace (Sheets, Drive, Gmail) for document creation and tracking.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Telegram URL Reception | Captures incoming Telegram messages as the workflow entry point |
| 2 | URL Analysis & Validation | AI parses the message, extracts and normalizes a job URL, confirms with the user, and routes valid links forward |
| 3 | Job Data Extraction | Scrapes the job page with Apify and retrieves the structured dataset |
| 4 | Data Formatting | Transforms raw scraped data into a unified text block for AI processing |
| 5 | Job Compatibility Evaluation | AI agent evaluates job-fit by querying resume vectors from Pinecone |
| 6 | Fit Outcome Determination | Parses the AI output, routes by fit score (≥70 = fit, <70 = not fit), sends rejection if needed |
| 7 | Job Approval Process | Sends an approval request to the user via Telegram and routes based on their response |
| 8 | Application Material Creation | AI generates cover letter, email, and improvement suggestions; updates user on progress |
| 9 | Job Tracking | Appends a structured row to Google Sheets |
| 10 | Final Application Steps | Creates a cover letter file in Google Drive, drafts an HR email in Gmail, and sends a final Telegram notification |
| 11 | Resume Vectorization Setup (Miscellaneous) | One-time/offline setup branch that downloads the resume from Google Drive, splits it, embeds it, and inserts vectors into Pinecone |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Telegram URL Reception

**Overview:** This block is the single entry point of the workflow. It listens for incoming Telegram messages and passes the raw message payload to the AI analysis block.

**Nodes Involved:** Telegram URL Trigger

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Telegram URL Trigger | `n8n-nodes-base.telegramTrigger` (v1.2) | Webhook-based trigger that fires on every incoming Telegram message | Updates filter set to `message` only; no additional fields | `{{ $json.message.text }}` used downstream | External Telegram API | Analyze URL Contents | Credential failure if Telegram Bot API token is invalid; no messages received if bot is not added to the chat; webhook registration may fail if n8n is not publicly reachable |

---

#### Block 2 — URL Analysis & Validation

**Overview:** An AI agent reads the incoming Telegram message, determines whether it contains a job link, normalizes the URL, and returns structured JSON. The result is parsed, a confirmation message is sent to the user, and a Switch node routes only valid links to the scraping stage.

**Nodes Involved:** Analyze URL Contents, OpenAI URL Parsing Agent, Construct Input Variables, Send URL Confirmation on Telegram, Switch by URL Validity

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Analyze URL Contents | `@n8n/n8n-nodes-langchain.agent` (v3.1) | LangChain Agent that processes the user's Telegram text | `text` = `{{ $json.message.text }}`; `promptType` = define; detailed system message instructing JSON-only output with fields `job_link`, `has_job_link`, `reply_message` | `$json.message.text` | Telegram URL Trigger | Construct Input Variables | If OpenAI returns non-JSON or malformed JSON, the downstream Code node handles it via try/catch |
| OpenAI URL Parsing Agent | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) | LLM sub-node providing language model to the Agent | Model = `gpt-5-mini`; no built-in tools; requires OpenAI API credential | — | Connected as `ai_languageModel` to Analyze URL Contents | — | API key exhaustion, model name typo (gpt-5-mini may not exist yet), rate limiting |
| Construct Input Variables | `n8n-nodes-base.code` (v2) | Parses the agent's output (string or object) into clean JSON; handles parse errors gracefully | JS code: tries `JSON.parse` on `raw` if string; on error returns fallback with `has_job_link: false` | `$json.output` | Analyze URL Contents | Send URL Confirmation on Telegram | If agent output is deeply nested or has `.content` wrapper, this node may not extract correctly (only checks `$json.output`) |
| Send URL Confirmation on Telegram | `n8n-nodes-base.telegram` (v1) | Sends the AI's reply_message back to the user's Telegram chat | `text` = `{{ $json.reply_message }}`; `chatId` = `{{ $('Telegram URL Trigger').item.json.message.chat.id }}` | References Telegram URL Trigger for chat ID | Construct Input Variables | Switch by URL Validity | Chat ID may be undefined if the trigger message structure changes |
| Switch by URL Validity | `n8n-nodes-base.switch` (v3.4) | Routes flow: Output 0 if `has_job_link` is true → proceed to scraping | Single rule: boolean check `{{ $('Construct Input Variables').item.json.has_job_link }}` is true | — | Send URL Confirmation on Telegram | Extract Content with Apify (output 0) | No output 1 defined — if `has_job_link` is false, the workflow silently stops (no explicit rejection at this stage) |

---

#### Block 3 — Job Data Extraction

**Overview:** This block uses Apify's Web Scraper actor to visit the job URL and extract page content (title, headings, first paragraph). A second Apify node retrieves the dataset results.

**Nodes Involved:** Extract Content with Apify, Retrieve Job Content with Apify

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Extract Content with Apify | `@apify/n8n-nodes-apify.apify` (v1) | Runs the `apify~web-scraper` actor on the job URL | Actor ID = `apify~web-scraper`; `actorSource` = store; `authentication` = apifyOAuth2Api; custom body includes `startUrls` with `url: $json.job_link`; `pageFunction` extracts `pageTitle`, `h1`, `first_h2`, `random_text_from_the_page`; proxy enabled; `runMode` = PRODUCTION; `waitUntil` = `networkidle2`; excludes `/**/*.{png,jpg,jpeg,pdf}` | `$json.job_link` | Switch by URL Validity (output 0) | Retrieve Job Content with Apify | Actor timeout; job URL behind login/JS-heavy page; Apify credit exhaustion; proxy failures; `defaultDatasetId` not returned on failure |
| Retrieve Job Content with Apify | `@apify/n8n-nodes-apify.apify` (v1) | Reads the dataset produced by the actor run | Resource = Datasets; `datasetId` = `{{ $json.defaultDatasetId }}` | `$json.defaultDatasetId` | Extract Content with Apify | Transform Job Data Format | Dataset may be empty if scraping failed; `defaultDatasetId` may be missing if actor errored |

---

#### Block 4 — Data Formatting

**Overview:** Transforms the raw Apify dataset output (which may contain nested objects and arrays) into a flat, concatenated text block (`unified_job_text`) that the AI agent can process.

**Nodes Involved:** Transform Job Data Format

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Transform Job Data Format | `n8n-nodes-base.code` (v2) | Recursively stringifies all values in the Apify dataset item; concatenates key-value pairs into `unified_job_text` | JS code: `stringifyValue()` helper handles null, string, number, boolean, array, object; output fields: `source_url`, `source_title`, `unified_job_text` | `$json.url`, `$json.title` | Retrieve Job Content with Apify | Assess Job-Fit Suitability | Very large pages may produce excessively long `unified_job_text` exceeding LLM context; special characters may need escaping |

---

#### Block 5 — Job Compatibility Evaluation

**Overview:** An AI agent evaluates whether the candidate fits the job by comparing the job description against resume data retrieved from a Pinecone vector store. The agent returns a structured fit report with score, reasons, and next-step directive.

**Nodes Involved:** Assess Job-Fit Suitability, GPT-5 Chat for Compatibility, Save Detailed Resume to Pinecone, Produce Embeddings for Resume

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Assess Job-Fit Suitability | `@n8n/n8n-nodes-langchain.agent` (v3.1) | LangChain Agent that evaluates job fit using resume context | `text` = `{{ $json.unified_job_text }}`; system message defines fit-score rules (≥70 = fit), JSON-only output format with fields: `is_fit`, `fit_score`, `job_role`, `company`, `main_reasons_for_fit`, `main_reasons_not_fit`, `short_user_message`, `next_step` | `$json.unified_job_text` | Transform Job Data Format | Compile Fit Analysis Report | If Pinecone returns no relevant vectors, agent may produce low-quality output; model may hallucinate skills not in resume |
| GPT-5 Chat for Compatibility | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) | LLM sub-node for the fit-evaluation agent | Model = `gpt-5-mini`; OpenAI API credential required | — | Connected as `ai_languageModel` to Assess Job-Fit Suitability | — | Same as other OpenAI nodes |
| Save Detailed Resume to Pinecone | `@n8n/n8n-nodes-langchain.vectorStorePinecone` (v1.3) | Pinecone vector store in `retrieve-as-tool` mode, exposed as a tool to the agent | Mode = retrieve-as-tool; index = `resume-profile`; namespace = `resume-profile`; `toolDescription` = "Work with the data in vector store for analysing the query by the agent" | — | Connected as `ai_tool` to Assess Job-Fit Suitability | — | Pinecone index must exist and contain vectors; namespace mismatch returns empty results; credential/region issues |
| Produce Embeddings for Resume | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` (v1.2) | OpenAI embeddings sub-node for the Pinecone tool | Default options; OpenAI API credential required | — | Connected as `ai_embedding` to Save Detailed Resume to Pinecone | — | Embedding model mismatch between insert and retrieve phases |

---

#### Block 6 — Fit Outcome Determination

**Overview:** Parses the AI agent's output into clean JSON, then uses a Switch node to branch: if `is_fit` is true, the flow proceeds to approval; if false, a rejection message is sent via Telegram.

**Nodes Involved:** Compile Fit Analysis Report, Decide on Fit Status, Send Rejection Notice on Telegram

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Compile Fit Analysis Report | `n8n-nodes-base.code` (v2) | Parses the agent output (handles string, `.content` wrapper, or raw object) into JSON | JS code: tries `$json.output`, `$json.text`, `$json.response`, `$json`; attempts `JSON.parse` on string; checks `raw?.content` | `$json.output`, `$json.text`, `$json.response`, `$json.content` | Assess Job-Fit Suitability | Decide on Fit Status | If the agent output format changes, this parser may fail; no schema validation |
| Decide on Fit Status | `n8n-nodes-base.switch` (v3.4) | Routes by `is_fit` boolean | Output 0: `is_fit` equals `true` → Seek Job Approval; Output 1: `is_fit` is false → Send Rejection | `{{ $json.is_fit }}` | Compile Fit Analysis Report | Seek Job Approval via Telegram (0), Send Rejection Notice on Telegram (1) | If `is_fit` is undefined or not boolean, routing may fail |
| Send Rejection Notice on Telegram | `n8n-nodes-base.telegram` (v1) | Sends a Telegram message with the fit score and explanation | `text` = fit score + short_user_message; `chatId` = `{{ $('Receive Job Input').item.json.message.chat.id }}` | References "Receive Job Input" (which is the Telegram URL Trigger by internal alias) | Decide on Fit Status (output 1) | — (terminal) | If "Receive Job Input" node name doesn't match exactly, chatId reference breaks; note: the workflow references "Receive Job Input" but the actual trigger node is named "Telegram URL Trigger" — this is a **critical name mismatch** that will cause a runtime error |

---

#### Block 7 — Job Approval Process

**Overview:** When the candidate is a fit, the workflow sends an approval request via Telegram (using `sendAndWait` for interactive buttons). The user can approve or skip. A Switch node routes accordingly: approved → proceed to application generation; skipped → send acknowledgment.

**Nodes Involved:** Seek Job Approval via Telegram, Route by Approval Status, Send Acknowledgment Message

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Seek Job Approval via Telegram | `n8n-nodes-base.telegram` (v1) | Sends an interactive approval message using Telegram's sendAndWait operation | Operation = `sendAndWait`; `approvalType` = double (approve/disapprove); `disapproveLabel` = "❌ Skip"; `chatId` = `{{ $('Receive Job Input').item.json.message.chat.id }}`; message includes fit_score and short_user_message | References "Receive Job Input" for chat ID | Decide on Fit Status (output 0) | Route by Approval Status | Same "Receive Job Input" naming issue as above; `sendAndWait` requires the workflow to stay active and wait for user response — timeout if user doesn't respond; Telegram inline keyboard may not render on some clients |
| Route by Approval Status | `n8n-nodes-base.switch` (v3.4) | Routes by `data.approved` boolean from the sendAndWait response | Output 0: `approved` = true → Telegram Status Update; Output 1: `approved` = false → Send Acknowledgment | `{{ $json.data.approved }}` | Seek Job Approval via Telegram | Telegram Status Update (0), Send Acknowledgment Message (1) | If the response structure changes (e.g., `data.approved` field name), routing breaks |
| Send Acknowledgment Message | `n8n-nodes-base.telegram` (v1) | Sends "Noted, Please share another job link" when user skips | Static text; `chatId` = `{{ $('Receive Job Input').item.json.message.chat.id }}` | — | Route by Approval Status (output 1) | — (terminal) | Same chat ID reference issue |

---

#### Block 8 — Application Material Creation

**Overview:** Once approved, an AI agent generates a complete application kit: cover letter, HR email, resume improvement suggestions, and metadata. The agent uses resume context from Pinecone. The output is parsed into structured JSON for downstream use.

**Nodes Involved:** Telegram Status Update, Prepare Application Kit, GPT-5 Chat for Application Kit, Save Resume Data to Pinecone, Generate Resume Embeddings, Create Material Summary Report

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Telegram Status Update | `n8n-nodes-base.telegram` (v1) | Sends a progress message to the user confirming the application process has started | Static text listing steps (tracker, resume, cover letter, email); `chatId` references "Receive Job Input" | — | Route by Approval Status (output 0) | Prepare Application Kit | Same chat ID reference issue |
| Prepare Application Kit | `@n8n/n8n-nodes-langchain.agent` (v3.1) | LangChain Agent that generates cover letter, HR email, resume suggestions, and metadata | `text` = `{{ $('Normalize Job Data').item.json.unified_job_text }}`; extensive system message defining JSON output format with fields: `url`, `company`, `role`, `location`, `fit_score`, `approval_status`, `candidate_name`, `cover_letter`, `hr_email_subject`, `hr_email_body`, `resume_improvement_suggestions`, `user_message`; references `$('Compile Fit Analysis Report')` and `$('Receive Job Input')` for input data | `$json.source_url`, `$('Compile Fit Analysis Report').item.json.company`, `$('Compile Fit Analysis Report').item.json.job_role`, `$('Compile Fit Analysis Report').item.json.fit_score`, `$('Receive Job Input').item.json.message.chat.first_name/last_name` | Telegram Status Update | Create Material Summary Report | The system message references "Normalize Job Data" node which does not exist in the workflow (the correct node is "Transform Job Data Format") — this is a **critical reference error**; cover letter word count may vary; candidate name may fall back to Telegram first/last name |
| GPT-5 Chat for Application Kit | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) | LLM sub-node for the application-kit agent | Model = `gpt-5-mini`; OpenAI API credential | — | Connected as `ai_languageModel` to Prepare Application Kit | — | Same as other OpenAI nodes |
| Save Resume Data to Pinecone | `@n8n/n8n-nodes-langchain.vectorStorePinecone` (v1.3) | Pinecone vector store in `retrieve-as-tool` mode for the application-kit agent | Mode = retrieve-as-tool; index = `resume-profile`; namespace = `resume-profile`; `toolDescription` = "Work with the data in pinecone vector store to provide answer the the queries" | — | Connected as `ai_tool` to Prepare Application Kit | — | Typo in toolDescription ("the the"); same Pinecone caveats |
| Generate Resume Embeddings | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` (v1.2) | OpenAI embeddings sub-node for the Pinecone tool | Default options; OpenAI API credential | — | Connected as `ai_embedding` to Save Resume Data to Pinecone | — | Same as other embeddings nodes |
| Create Material Summary Report | `n8n-nodes-base.code` (v2) | Parses the application-kit agent output into clean JSON (same parser as Compile Fit Analysis Report) | JS code: handles string, `.content` wrapper, raw object | `$json.output`, `$json.text`, `$json.response`, `$json.content` | Prepare Application Kit | Add Entry to Job Tracker Sheets | Same parsing fragility as Compile Fit Analysis Report |

---

#### Block 9 — Job Tracking

**Overview:** Appends a structured row to a Google Sheets spreadsheet ("Application Tracker" → sheet "Jobs") containing all relevant job and application data.

**Nodes Involved:** Add Entry to Job Tracker Sheets

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Add Entry to Job Tracker Sheets | `n8n-nodes-base.googleSheets` (v4.7) | Appends a row to Google Sheets | Operation = append; document = "Application Tracker"; sheet = "Jobs"; columns: Company, Role, Job Url, Location, Fit Score, Approval Status, Resume Improvement Suggestions; `mappingMode` = defineBelow; all values pulled from earlier nodes | `$('Compile Fit Analysis Report').item.json.job_role`, `.company`, `.fit_score`; `$json.url`, `$json.location`, `$json.approval_status`, `$json.resume_improvement_suggestions` | Create Material Summary Report | Draft Cover Letter in Google Drive | Google Sheets credential must have write access; sheet must have matching column headers; `resume_improvement_suggestions` is an array — Sheets may not handle arrays well (may need `.join()`); sheet ID/document ID must be correctly configured |

---

#### Block 10 — Final Application Steps

**Overview:** Creates a cover letter text file in Google Drive, drafts an email to HR in Gmail, and sends a final notification to the user on Telegram.

**Nodes Involved:** Draft Cover Letter in Google Drive, Compose Email to HR, Telegram Final Job Notification

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Draft Cover Letter in Google Drive | `n8n-nodes-base.googleDrive` (v3) | Creates a text file in Google Drive with the cover letter content | Operation = createFromText; name = "cover_letter"; content = `{{ $('Create Material Summary Report').item.json.cover_letter }}`; folder = "Cover Letter"; drive = "My Drive" | References Create Material Summary Report | Add Entry to Job Tracker Sheets | Compose Email to HR | Google Drive credential must have write access to the specified folder; folder must exist; file created as plain text (not .docx) |
| Compose Email to HR | `n8n-nodes-base.gmail` (v2.2) | Creates a draft email in Gmail (does not send) | Resource = draft; subject = `{{ $('Create Material Summary Report').item.json.hr_email_subject }}`; message = `{{ $('Create Material Summary Report').item.json.hr_email_body }}` | References Create Material Summary Report | Draft Cover Letter in Google Drive | Telegram Final Job Notification | Gmail credential must have compose scope; draft is created but not sent — user must review and send manually; recipient ("To") field is not set in configuration — **this will create a draft with no recipient**, which is a potential oversight |
| Telegram Final Job Notification | `n8n-nodes-base.telegram` (v1) | Sends the final user_message to the user on Telegram | `text` = `{{ $('Create Material Summary Report').item.json.user_message }}`; `chatId` = `{{ $('Receive Job Input').item.json.message.chat.id }}` | References Create Material Summary Report and Receive Job Input | Compose Email to HR | — (terminal) | Same chat ID reference issue |

---

#### Block 11 — Resume Vectorization Setup (Miscellaneous)

**Overview:** This is a standalone initialization branch that downloads the candidate's resume from Google Drive, splits it into chunks, generates OpenAI embeddings, and inserts the vectors into the Pinecone `resume-profile` index. This branch must be run once (or whenever the resume is updated) before the main workflow is triggered.

**Nodes Involved:** Download File from Google Drive, Save Profile Vectors to Pinecone, Create OpenAI Embeddings, Load Document Data, Recursive Text Splitter

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|---------------|---------------|---------------------------|-------|--------|------------|
| Download File from Google Drive | `n8n-nodes-base.googleDrive` (v3) | Downloads the resume file (my_resume.docx) from Google Drive | Operation = download; file = "my_resume.docx" | — | Manual trigger or external | Save Profile Vectors to Pinecone | File not found; credential lacks read access; .docx format may need conversion (binary output) |
| Save Profile Vectors to Pinecone | `@n8n/n8n-nodes-langchain.vectorStorePinecone` (v1.3) | Pinecone vector store in `insert` mode to upsert resume vectors | Mode = insert; index = `resume-profile`; namespace = `resume-profile` | — | Download File from Google Drive (main), Create OpenAI Embeddings (ai_embedding), Load Document Data (ai_document) | — | Pinecone index must exist; dimensionality must match the OpenAI embedding model; upsert rate limits |
| Create OpenAI Embeddings | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` (v1.2) | Generates OpenAI embeddings for the resume chunks | Default options; OpenAI API credential | — | Connected as `ai_embedding` to Save Profile Vectors to Pinecone | — | Same as other embeddings nodes |
| Load Document Data | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` (v1.1) | Loads binary document data and prepares it for vectorization | `dataType` = binary; `textSplittingMode` = custom | — | Recursive Text Splitter (ai_textSplitter) → Load Document Data → Save Profile Vectors to Pinecone (ai_document) | — | Binary data must be present from the Google Drive download; .docx parsing may fail if the loader doesn't support the format |
| Recursive Text Splitter | `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter` (v1) | Splits the resume text into chunks for embedding | Default options (no custom chunk size/overlap configured) | — | Connected as `ai_textSplitter` to Load Document Data | — | Default chunk size may split mid-sentence; for a short resume, splitting may not be necessary |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Telegram URL Trigger | telegramTrigger (v1.2) | Entry point — listens for Telegram messages | — | Analyze URL Contents | Telegram URL reception — Triggers when a URL is received via Telegram. |
| Analyze URL Contents | @n8n/n8n-nodes-langchain.agent (v3.1) | AI agent parses message and extracts job link | Telegram URL Trigger | Construct Input Variables | URL analysis and validation — AI analyzes the URL to build and confirm job data inputs. |
| OpenAI URL Parsing Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | LLM sub-node for URL parsing agent | — (sub-node of Analyze URL Contents) | — (sub-node) | URL analysis and validation — AI analyzes the URL to build and confirm job data inputs. |
| Construct Input Variables | n8n-nodes-base.code (v2) | Parses agent JSON output into clean variables | Analyze URL Contents | Send URL Confirmation on Telegram | URL analysis and validation — AI analyzes the URL to build and confirm job data inputs. |
| Send URL Confirmation on Telegram | n8n-nodes-base.telegram (v1) | Sends AI reply back to user on Telegram | Construct Input Variables | Switch by URL Validity | URL analysis and validation — AI analyzes the URL to build and confirm job data inputs. |
| Switch by URL Validity | n8n-nodes-base.switch (v3.4) | Routes valid job links to scraping | Send URL Confirmation on Telegram | Extract Content with Apify | URL analysis and validation — AI analyzes the URL to build and confirm job data inputs. |
| Extract Content with Apify | @apify/n8n-nodes-apify.apify (v1) | Runs Apify web scraper on the job URL | Switch by URL Validity | Retrieve Job Content with Apify | Job data extraction — Extracts job content if the URL is valid. |
| Retrieve Job Content with Apify | @apify/n8n-nodes-apify.apify (v1) | Reads scraped dataset from Apify | Extract Content with Apify | Transform Job Data Format | Data formatting and evaluation — Formats the extracted data and evaluates job compatibility using AI. |
| Transform Job Data Format | n8n-nodes-base.code (v2) | Converts scraped data into unified text block | Retrieve Job Content with Apify | Assess Job-Fit Suitability | Data formatting and evaluation — Formats the extracted data and evaluates job compatibility using AI. |
| Assess Job-Fit Suitability | @n8n/n8n-nodes-langchain.agent (v3.1) | AI agent evaluates candidate-job fit | Transform Job Data Format | Compile Fit Analysis Report | Job compatibility evaluation — AI evaluates the job compatibility, generating a fit report. |
| GPT-5 Chat for Compatibility | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | LLM sub-node for fit-evaluation agent | — (sub-node) | — (sub-node) | Job compatibility evaluation — AI evaluates the job compatibility, generating a fit report. |
| Save Detailed Resume to Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone (v1.3) | Pinecone tool for resume retrieval in fit evaluation | — (sub-node) | — (sub-node of Assess Job-Fit Suitability) | Job compatibility evaluation — AI evaluates the job compatibility, generating a fit report. |
| Produce Embeddings for Resume | @n8n/n8n-nodes-langchain.embeddingsOpenAi (v1.2) | Embeddings sub-node for Pinecone tool in fit evaluation | — (sub-node) | — (sub-node) | Job compatibility evaluation — AI evaluates the job compatibility, generating a fit report. |
| Compile Fit Analysis Report | n8n-nodes-base.code (v2) | Parses fit-evaluation agent output into JSON | Assess Job-Fit Suitability | Decide on Fit Status | Fit outcome determination — Decides the outcome based on the fit report and notifies the rejection via Telegram. |
| Decide on Fit Status | n8n-nodes-base.switch (v3.4) | Routes by is_fit boolean | Compile Fit Analysis Report | Seek Job Approval via Telegram (0), Send Rejection Notice on Telegram (1) | Fit outcome determination — Decides the outcome based on the fit report and notifies the rejection via Telegram. |
| Send Rejection Notice on Telegram | n8n-nodes-base.telegram (v1) | Sends rejection message with fit score | Decide on Fit Status (output 1) | — (terminal) | Fit outcome determination — Decides the outcome based on the fit report and notifies the rejection via Telegram. |
| Seek Job Approval via Telegram | n8n-nodes-base.telegram (v1) | Sends interactive approval request via Telegram | Decide on Fit Status (output 0) | Route by Approval Status | Job approval process — Sends and directs approval requests based on evaluation. |
| Route by Approval Status | n8n-nodes-base.switch (v3.4) | Routes by user approval response | Seek Job Approval via Telegram | Telegram Status Update (0), Send Acknowledgment Message (1) | Job approval process — Sends and directs approval requests based on evaluation. |
| Send Acknowledgment Message | n8n-nodes-base.telegram (v1) | Sends "share another link" message on skip | Route by Approval Status (output 1) | — (terminal) | Job approval process — Sends and directs approval requests based on evaluation. |
| Telegram Status Update | n8n-nodes-base.telegram (v1) | Sends progress notification to user | Route by Approval Status (output 0) | Prepare Application Kit | Application material creation — AI generates and manages application kits post-approval. |
| Prepare Application Kit | @n8n/n8n-nodes-langchain.agent (v3.1) | AI agent generates cover letter, email, and suggestions | Telegram Status Update | Create Material Summary Report | Application material creation — AI generates and manages application kits post-approval. |
| GPT-5 Chat for Application Kit | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | LLM sub-node for application-kit agent | — (sub-node) | — (sub-node) | Application material creation — AI generates and manages application kits post-approval. |
| Save Resume Data to Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone (v1.3) | Pinecone tool for resume retrieval in application kit agent | — (sub-node) | — (sub-node of Prepare Application Kit) | Application material creation — AI generates and manages application kits post-approval. |
| Generate Resume Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi (v1.2) | Embeddings sub-node for Pinecone tool in application kit | — (sub-node) | — (sub-node) | Application material creation — AI generates and manages application kits post-approval. |
| Create Material Summary Report | n8n-nodes-base.code (v2) | Parses application-kit agent output into JSON | Prepare Application Kit | Add Entry to Job Tracker Sheets | Application material creation — AI generates and manages application kits post-approval. |
| Add Entry to Job Tracker Sheets | n8n-nodes-base.googleSheets (v4.7) | Appends job data row to Google Sheets | Create Material Summary Report | Draft Cover Letter in Google Drive | Job tracking — Logs the application's journey into Google Sheets. |
| Draft Cover Letter in Google Drive | n8n-nodes-base.googleDrive (v3) | Creates cover letter text file in Google Drive | Add Entry to Job Tracker Sheets | Compose Email to HR | Final application steps — Concludes the application by creating a cover letter and notifying HR via email and Telegram. |
| Compose Email to HR | n8n-nodes-base.gmail (v2.2) | Creates draft email in Gmail | Draft Cover Letter in Google Drive | Telegram Final Job Notification | Final application steps — Concludes the application by creating a cover letter and notifying HR via email and Telegram. |
| Telegram Final Job Notification | n8n-nodes-base.telegram (v1) | Sends final completion message to user | Compose Email to HR | — (terminal) | Final application steps — Concludes the application by creating a cover letter and notifying HR via email and Telegram. |
| Download File from Google Drive | n8n-nodes-base.googleDrive (v3) | Downloads resume file for vectorization | — (manual/separate trigger) | Save Profile Vectors to Pinecone | Miscellaneous — Handles default data loading, Google Drive operations and initial setups. |
| Save Profile Vectors to Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone (v1.3) | Inserts resume vectors into Pinecone | Download File from Google Drive, Create OpenAI Embeddings, Load Document Data | — | Miscellaneous — Handles default data loading, Google Drive operations and initial setups. |
| Create OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi (v1.2) | Generates embeddings for resume vectorization | — (sub-node) | — (sub-node) | Miscellaneous — Handles default data loading, Google Drive operations and initial setups. |
| Load Document Data | @n8n/n8n-nodes-langchain.documentDefaultDataLoader (v1.1) | Loads binary resume data for processing | Recursive Text Splitter (ai_textSplitter) | Save Profile Vectors to Pinecone (ai_document) | Miscellaneous — Handles default data loading, Google Drive operations and initial setups. |
| Recursive Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter (v1) | Splits resume text into chunks | — (sub-node) | Load Document Data (ai_textSplitter) | Miscellaneous — Handles default data loading, Google Drive operations and initial setups. |

---

### 4. Reproducing the Workflow from Scratch

**Prerequisites (credentials to configure before building):**
- **Telegram Bot API** — Create a bot via BotFather, obtain the API token
- **OpenAI API** — Obtain an API key with access to the `gpt-5-mini` model (or substitute with an available model like `gpt-4o-mini`)
- **Apify OAuth2** — Create an Apify account, generate an API token with access to the `apify~web-scraper` actor
- **Pinecone** — Create a project, create an index named `resume-profile` (dimension must match OpenAI embeddings, typically 1536 for `text-embedding-ada-002` or 3072 for `text-embedding-3-large`), obtain API key and environment
- **Google Sheets** — OAuth2 credential with write access; create a spreadsheet named "Application Tracker" with a sheet named "Jobs" containing columns: Company, Role, Job Url, Location, Fit Score, Approval Status, Resume Improvement Suggestions
- **Google Drive** — OAuth2 credential with write access; create a folder named "Cover Letter"; upload a resume file named "my_resume.docx"
- **Gmail** — OAuth2 credential with compose/draft scope

---

**Step-by-step rebuild:**

1. **Create the workflow** in n8n and name it "Evaluate job fit and generate application assets from Telegram links with OpenAI, Pinecone, Apify and Google Sheets".

2. **Add Telegram URL Trigger:**
   - Node type: `Telegram Trigger`
   - Version: 1.2
   - Updates filter: `message`
   - Credential: select your Telegram Bot API credential
   - Note the webhook ID (auto-generated)

3. **Add Analyze URL Contents:**
   - Node type: `AI Agent`
   - Version: 3.1
   - Prompt type: Define
   - Text: `={{ $json.message.text }}`
   - System message: Full prompt instructing JSON-only output with fields `job_link`, `has_job_link`, `reply_message`, URL normalization rules, and polite reply requirements
   - Connect: Telegram URL Trigger → Analyze URL Contents (main)

4. **Add OpenAI URL Parsing Agent (sub-node):**
   - Node type: `OpenAI Chat Model`
   - Version: 1.3
   - Model: `gpt-5-mini`
   - Credential: select your OpenAI API credential
   - Connect: OpenAI URL Parsing Agent → Analyze URL Contents (ai_languageModel)

5. **Add Construct Input Variables:**
   - Node type: `Code`
   - Version: 2
   - JS code: Parse `$json.output` with try/catch for JSON.parse; on failure return `{ job_link: '', has_job_link: false, reply_message: 'Please send a valid job link...', parse_error: ..., raw_output: ... }`
   - Connect: Analyze URL Contents → Construct Input Variables (main)

6. **Add Send URL Confirmation on Telegram:**
   - Node type: `Telegram`
   - Version: 1
   - Text: `={{ $json.reply_message }}`
   - Chat ID: `={{ $('Telegram URL Trigger').item.json.message.chat.id }}`
   - Credential: same Telegram Bot API credential
   - Connect: Construct Input Variables → Send URL Confirmation on Telegram (main)

7. **Add Switch by URL Validity:**
   - Node type: `Switch`
   - Version: 3.4
   - Rule 0 (output 0): Boolean condition where `{{ $('Construct Input Variables').item.json.has_job_link }}` is `true`
   - Connect: Send URL Confirmation on Telegram → Switch by URL Validity (main)

8. **Add Extract Content with Apify:**
   - Node type: `Apify`
   - Version: 1
   - Actor ID: `apify~web-scraper` (from store)
   - Authentication: Apify OAuth2
   - Custom body: JSON object with `startUrls: [{ url: $json.job_link }]`, `pageFunction` extracting `pageTitle`, `h1`, `first_h2`, `random_text_from_the_page`, `proxyConfiguration: { useApifyProxy: true }`, `runMode: PRODUCTION`, `waitUntil: ['networkidle2']`, excludes pattern `/**/*.{png,jpg,jpeg,pdf}`, `injectJQuery: true`, `headless: true`
   - Connect: Switch by URL Validity (output 0) → Extract Content with Apify (main)

9. **Add Retrieve Job Content with Apify:**
   - Node type: `Apify`
   - Version: 1
   - Resource: Datasets
   - Dataset ID: `={{ $json.defaultDatasetId }}`
   - Authentication: Apify OAuth2
   - Connect: Extract Content with Apify → Retrieve Job Content with Apify (main)

10. **Add Transform Job Data Format:**
    - Node type: `Code`
    - Version: 2
    - JS code: Recursive `stringifyValue()` function; iterate over `$json` entries; concatenate into `unified_job_text`; output `source_url`, `source_title`, `unified_job_text`
    - Connect: Retrieve Job Content with Apify → Transform Job Data Format (main)

11. **Add Assess Job-Fit Suitability:**
    - Node type: `AI Agent`
    - Version: 3.1
    - Prompt type: Define
    - Text: `={{ $json.unified_job_text }}`
    - System message: Full prompt for job-fit evaluator with fit_score ≥70 threshold, JSON-only output with fields `is_fit`, `fit_score`, `job_role`, `company`, `main_reasons_for_fit`, `main_reasons_not_fit`, `short_user_message`, `next_step`
    - Connect: Transform Job Data Format → Assess Job-Fit Suitability (main)

12. **Add GPT-5 Chat for Compatibility (sub-node):**
    - Node type: `OpenAI Chat Model`
    - Version: 1.3
    - Model: `gpt-5-mini`
    - Credential: OpenAI API
    - Connect: GPT-5 Chat for Compatibility → Assess Job-Fit Suitability (ai_languageModel)

13. **Add Save Detailed Resume to Pinecone (sub-node):**
    - Node type: `Pinecone Vector Store`
    - Version: 1.3
    - Mode: retrieve-as-tool
    - Index: `resume-profile`
    - Namespace: `resume-profile`
    - Tool description: "Work with the data in vector store for analysing the query by the agent"
    - Connect: Save Detailed Resume to Pinecone → Assess Job-Fit Suitability (ai_tool)

14. **Add Produce Embeddings for Resume (sub-node):**
    - Node type: `OpenAI Embeddings`
    - Version: 1.2
    - Credential: OpenAI API
    - Connect: Produce Embeddings for Resume → Save Detailed Resume to Pinecone (ai_embedding)

15. **Add Compile Fit Analysis Report:**
    - Node type: `Code`
    - Version: 2
    - JS code: Parse `$json.output` / `$json.text` / `$json.response` / `$json` with JSON.parse for strings; handle `.content` wrapper
    - Connect: Assess Job-Fit Suitability → Compile Fit Analysis Report (main)

16. **Add Decide on Fit Status:**
    - Node type: `Switch`
    - Version: 3.4
    - Output 0: `is_fit` equals `true` → Seek Job Approval
    - Output 1: `is_fit` is `false` → Send Rejection
    - Connect: Compile Fit Analysis Report → Decide on Fit Status (main)

17. **Add Send Rejection Notice on Telegram:**
    - Node type: `Telegram`
    - Version: 1
    - Text: `={{ 'Your Fit Score for this job is ' + $('Compile Fit Analysis Report').item.json.fit_score + '\n' + $json.short_user_message }}`
    - Chat ID: `={{ $('Telegram URL Trigger').item.json.message.chat.id }}`
    - Credential: Telegram Bot API
    - Connect: Decide on Fit Status (output 1) → Send Rejection Notice on Telegram (main)

18. **Add Seek Job Approval via Telegram:**
    - Node type: `Telegram`
    - Version: 1
    - Operation: `sendAndWait`
    - Approval type: double
    - Disapprove label: "❌ Skip"
    - Chat ID: `={{ $('Telegram URL Trigger').item.json.message.chat.id }}`
    - Message: `={{ 'Your job fit-score for this role is ' + $json.fit_score + '!\n' + $json.short_user_message }}`
    - Credential: Telegram Bot API
    - Connect: Decide on Fit Status (output 0) → Seek Job Approval via Telegram (main)

19. **Add Route by Approval Status:**
    - Node type: `Switch`
    - Version: 3.4
    - Output 0: `$json.data.approved` equals `true` → Telegram Status Update
    - Output 1: `$json.data.approved` is `false` → Send Acknowledgment Message
    - Connect: Seek Job Approval via Telegram → Route by Approval Status (main)

20. **Add Send Acknowledgment Message:**
    - Node type: `Telegram`
    - Version: 1
    - Text: "Noted, Please share another job link"
    - Chat ID: `={{ $('Telegram URL Trigger').item.json.message.chat.id }}`
    - Credential: Telegram Bot API
    - Connect: Route by Approval Status (output 1) → Send Acknowledgment Message (main)

21. **Add Telegram Status Update:**
    - Node type: `Telegram`
    - Version: 1
    - Text: "Great 👍 Starting your application process for this role.\n\nI will now:\n- Create a tracker entry\n- Tailor your resume\n- Generate a cover letter\n- Draft an email for the recruiter\n\nI'll update you once everything is ready 🚀"
    - Chat ID: `={{ $('Telegram URL Trigger').item.json.message.chat.id }}`
    - Credential: Telegram Bot API
    - Connect: Route by Approval Status (output 0) → Telegram Status Update (main)

22. **Add Prepare Application Kit:**
    - Node type: `AI Agent`
    - Version: 3.1
    - Prompt type: Define
    - Text: `={{ $('Transform Job Data Format').item.json.unified_job_text }}`
    - System message: Full prompt for application material generation with JSON output fields (`url`, `company`, `role`, `location`, `fit_score`, `approval_status`, `candidate_name`, `cover_letter`, `hr_email_subject`, `hr_email_body`, `resume_improvement_suggestions`, `user_message`); references `$('Compile Fit Analysis Report')` for company/role/fit_score and `$('Telegram URL Trigger')` for candidate name fallback
    - Connect: Telegram Status Update → Prepare Application Kit (main)

23. **Add GPT-5 Chat for Application Kit (sub-node):**
    - Node type: `OpenAI Chat Model`
    - Version: 1.3
    - Model: `gpt-5-mini`
    - Credential: OpenAI API
    - Connect: GPT-5 Chat for Application Kit → Prepare Application Kit (ai_languageModel)

24. **Add Save Resume Data to Pinecone (sub-node):**
    - Node type: `Pinecone Vector Store`
    - Version: 1.3
    - Mode: retrieve-as-tool
    - Index: `resume-profile`
    - Namespace: `resume-profile`
    - Tool description: "Work with the data in pinecone vector store to provide answer the the queries"
    - Connect: Save Resume Data to Pinecone → Prepare Application Kit (ai_tool)

25. **Add Generate Resume Embeddings (sub-node):**
    - Node type: `OpenAI Embeddings`
    - Version: 1.2
    - Credential: OpenAI API
    - Connect: Generate Resume Embeddings → Save Resume Data to Pinecone (ai_embedding)

26. **Add Create Material Summary Report:**
    - Node type: `Code`
    - Version: 2
    - JS code: Same parser as Compile Fit Analysis Report — handles `$json.output`, `$json.text`, `$json.response`, `$json.content`
    - Connect: Prepare Application Kit → Create Material Summary Report (main)

27. **Add Add Entry to Job Tracker Sheets:**
    - Node type: `Google Sheets`
    - Version: 4.7
    - Operation: append
    - Document: "Application Tracker"
    - Sheet: "Jobs"
    - Columns mapped: Company ← `$('Compile Fit Analysis Report').item.json.company`, Role ← `$('Compile Fit Analysis Report').item.json.job_role`, Job Url ← `$json.url`, Location ← `$json.location`, Fit Score ← `$('Compile Fit Analysis Report').item.json.fit_score`, Approval Status ← `$json.approval_status`, Resume Improvement Suggestions ← `$json.resume_improvement_suggestions`
    - Credential: Google Sheets OAuth2
    - Connect: Create Material Summary Report → Add Entry to Job Tracker Sheets (main)

28. **Add Draft Cover Letter in Google Drive:**
    - Node type: `Google Drive`
    - Version: 3
    - Operation: createFromText
    - File name: "cover_letter"
    - Content: `={{ $('Create Material Summary Report').item.json.cover_letter }}`
    - Folder: "Cover Letter"
    - Drive: "My Drive"
    - Credential: Google Drive OAuth2
    - Connect: Add Entry to Job Tracker Sheets → Draft Cover Letter in Google Drive (main)

29. **Add Compose Email to HR:**
    - Node type: `Gmail`
    - Version: 2.2
    - Resource: draft
    - Subject: `={{ $('Create Material Summary Report').item.json.hr_email_subject }}`
    - Message: `={{ $('Create Material Summary Report').item.json.hr_email_body }}`
    - Credential: Gmail OAuth2
    - Connect: Draft Cover Letter in Google Drive → Compose Email to HR (main)

30. **Add Telegram Final Job Notification:**
    - Node type: `Telegram`
    - Version: 1
    - Text: `={{ $('Create Material Summary Report').item.json.user_message }}`
    - Chat ID: `={{ $('Telegram URL Trigger').item.json.message.chat.id }}`
    - Credential: Telegram Bot API
    - Connect: Compose Email to HR → Telegram Final Job Notification (main)

31. **Add Download File from Google Drive (standalone branch):**
    - Node type: `Google Drive`
    - Version: 3
    - Operation: download
    - File: "my_resume.docx"
    - Credential: Google Drive OAuth2
    - This node is not connected to the main flow; execute it manually to seed Pinecone

32. **Add Save Profile Vectors to Pinecone:**
    - Node type: `Pinecone Vector Store`
    - Version: 1.3
    - Mode: insert
    - Index: `resume-profile`
    - Namespace: `resume-profile`
    - Connect: Download File from Google Drive → Save Profile Vectors to Pinecone (main)

33. **Add Create OpenAI Embeddings (sub-node for insert Pinecone):**
    - Node type: `OpenAI Embeddings`
    - Version: 1.2
    - Credential: OpenAI API
    - Connect: Create OpenAI Embeddings → Save Profile Vectors to Pinecone (ai_embedding)

34. **Add Load Document Data (sub-node for insert Pinecone):**
    - Node type: `Default Data Loader`
    - Version: 1.1
    - Data type: binary
    - Text splitting mode: custom
    - Connect: Load Document Data → Save Profile Vectors to Pinecone (ai_document)

35. **Add Recursive Text Splitter (sub-node for Load Document Data):**
    - Node type: `Recursive Character Text Splitter`
    - Version: 1
    - Default options
    - Connect: Recursive Text Splitter → Load Document Data (ai_textSplitter)

36. **Execute the vectorization branch once:** Manually trigger the "Download File from Google Drive" node to download the resume, split it, embed it, and insert vectors into Pinecone. This only needs to run once (or whenever the resume changes).

37. **Activate the workflow** — ensure the n8n instance is publicly reachable so the Telegram webhook can be registered.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|----------------|
| **Critical naming issue:** Several nodes (Send Rejection Notice, Seek Job Approval, Send Acknowledgment, Telegram Status Update, Telegram Final Job Notification) reference `$('Receive Job Input')` for the chat ID, but the actual trigger node is named "Telegram URL Trigger". These expressions will fail at runtime unless a node named "Receive Job Input" exists (e.g., by renaming the trigger node). | Affects 5 Telegram nodes; rename the trigger node or update all references |
| **Critical node reference issue:** The Prepare Application Kit agent's system message references `$('Normalize Job Data')` which does not exist in the workflow. The correct reference should be `$('Transform Job Data Format')`. This will cause the agent to lack job data context. | Fix the system message expression before deploying |
| **Gmail draft missing recipient:** The Compose Email to HR node creates a draft with subject and body but no "To" field. The draft will have no recipient and must be manually addressed before sending. | Add a `toEmail` parameter or a step to look up the HR email |
| **Model availability:** The workflow specifies `gpt-5-mini` which may not be a publicly available model. Substitute with `gpt-4o-mini` or the appropriate model available in your OpenAI account. | Verify model availability in your OpenAI dashboard |
| **Switch by URL Validity has no fallback output:** When `has_job_link` is false, the workflow does not define an output branch. The flow silently stops. Consider adding a Telegram message or error handling for invalid input. | Enhance user experience for non-link messages |
| **Pinecone index dimension must match embeddings:** The OpenAI embeddings node uses the default model (likely `text-embedding-ada-002` at 1536 dimensions). Ensure the Pinecone index `resume-profile` is created with the same dimensionality. | Pinecone index configuration |
| **Array in Google Sheets:** The `resume_improvement_suggestions` field is an array. Google Sheets may not handle arrays natively — consider joining with `.join('\n')` in the expression. | Data formatting for Sheets |
| **Resume vectorization is a one-time setup:** The Miscellaneous block (nodes 31–35) must be executed before the main workflow to seed Pinecone with resume vectors. It is not part of the automated Telegram flow. | Run once before activating the workflow |
| **Cover letter format:** The Draft Cover Letter in Google Drive node creates a plain text file, not a .docx. If a formatted document is needed, consider using a Google Docs template or a document conversion step. | Document format consideration |
| **Overview sticky note:** Full workflow description including how it works, setup steps, and customization tips is embedded in the workflow as Sticky Note 16. | Refer to the sticky note for a quick summary within n8n |