Write personalized cold emails from LinkedIn leads with Wiza, Perplexity and GPT-5

https://n8nworkflows.xyz/workflows/write-personalized-cold-emails-from-linkedin-leads-with-wiza--perplexity-and-gpt-5-12342


# Write personalized cold emails from LinkedIn leads with Wiza, Perplexity and GPT-5

## 1. Workflow Overview

**Title:** *Write personalized cold emails from LinkedIn leads with Wiza, Perplexity and GPT-5*  
**Purpose:** Collect a LinkedIn profile URL, enrich the lead with Wiza (email + company data), research the lead/company on the web via Perplexity, then use GPT-5 to generate a personalized cold email and create a **Gmail draft** for review.

**Primary use case:** Outbound sales teams/founders who source leads from LinkedIn (specifically companies hiring SDR/BDR roles) and want fast, consistent personalization at scale while keeping sending manual (draft-first).

### 1.1 Input Reception & Lead Enrichment (Wiza)
Form intake → Wiza enrichment → route on success/failure.

### 1.2 Lead Normalization & Storage (Data Table)
Map Wiza output into a stable schema → store in “Leads” data table.

### 1.3 Prospect & Company Research (Perplexity + GPT)
Use Perplexity as a tool inside an AI Agent to compile a research report grounded in the “SDR/BDR hiring” intent signal.

### 1.4 Email Personalization (Case Studies + GPT-5 + Structured JSON)
Use GPT-5 Agent with strict style rules and a structured output parser to produce a short subject + HTML body.

### 1.5 Email Staging (Gmail Draft)
Create a Gmail draft addressed to the enriched email.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Lead Enrichment (Wiza)

**Overview:** Captures a LinkedIn URL from a form, sends it to Wiza for enrichment, then branches depending on whether Wiza returns `status=finished` or `status=failed`.

**Nodes involved:**
- `On form submission1`
- `Email Finder`
- `Switch1`
- `Failed To Find Enrichment`

#### Node: On form submission1
- **Type / role:** `Form Trigger` — workflow entry point via n8n-hosted form.
- **Config (interpreted):**
  - Form title: **“Linkedin”**
  - One field: **Linkedin** (string)
- **Key variables:**
  - Output includes `{{$json.Linkedin}}`
- **Connections:**
  - Output → `Email Finder`
- **Failure modes / edge cases:**
  - Empty/invalid LinkedIn URL (Wiza may fail or return no enrichment).
  - If form is publicly accessible, spam submissions may consume Wiza/OpenAI credits.

#### Node: Email Finder
- **Type / role:** `Wiza` (`@wizaco/n8n-nodes-wiza.wiza`) — enrich lead from LinkedIn URL.
- **Config (interpreted):**
  - Input type: **linkedinUrl**
  - LinkedIn URL: `={{ $json.Linkedin }}`
- **Credentials:** Wiza API credential required.
- **Connections:**
  - Output → `Switch1`
- **Failure modes / edge cases:**
  - Wiza auth/plan limits, rate limits.
  - LinkedIn URL not resolvable / private profile / blocked.
  - Wiza may return partial data (missing email, domain, etc.) even when not “failed”.

#### Node: Switch1
- **Type / role:** `Switch` — routes by enrichment status.
- **Config (interpreted):**
  - Output **Failed** when `{{$json.status}} == "failed"`
  - Output **Finished** when `{{$json.status}} == "finished"`
- **Connections:**
  - `Failed` → `Failed To Find Enrichment`
  - `Finished` → `Lead Information`
- **Failure modes / edge cases:**
  - If Wiza returns a different status (e.g., `pending`) it will match neither rule and produce no downstream output (workflow appears to “stop”).
  - If `status` is missing, expression evaluates to empty and no branch matches.

#### Node: Failed To Find Enrichment
- **Type / role:** `Form` — completion page shown to the user when enrichment fails.
- **Config (interpreted):**
  - Operation: **completion**
  - Title: “Failed To Find Enrichment Data”
  - Message: “The Wiza Back-End Could Not Find any Enrichment Data from the provided information”
- **Connections:** none (terminal)
- **Failure modes / edge cases:**
  - This does not log/store the failed lead. If you want auditing, add a data table insert here.

---

### Block 2 — Lead Normalization & Storage (Data Table)

**Overview:** Converts Wiza enrichment fields into a consistent lead schema, then writes the record into an n8n Data Table (“Leads”).

**Nodes involved:**
- `Lead Information`
- `Add To lead Database`

#### Node: Lead Information
- **Type / role:** `Set` — remap fields from Wiza output to canonical names.
- **Config (interpreted):** Creates fields:
  - `full_name = {{$json.name}}`
  - `title = {{$json.title}}`
  - `location = {{$json.location}}`
  - `company_industry = {{$json.company_industry}}`
  - `company_domain = {{$json.company_domain}}`
  - `company_revenue = {{$json.company_revenue_range}}`
  - `company_size = {{$json.company_size}}`
  - `company_type = {{$json.company_type}}`
  - `company_description = {{$json.company_description}}`
  - `company_location = {{$json.company_location}}`
  - `email = {{$json.email}}`
  - `id = {{$json.id}}` (number)
- **Connections:**
  - Output → `Add To lead Database`
- **Failure modes / edge cases:**
  - If Wiza field names differ (API changes / different plan output), mappings may become undefined.
  - `id` type mismatch (string vs number) could cause issues if later nodes assume numeric.

#### Node: Add To lead Database
- **Type / role:** `Data Table` — insert/append a row into “Leads”.
- **Config (interpreted):**
  - Data Table: **Leads** (id `esfg4aP7UBYpkcGt`)
  - Columns written (mapping highlights):
    - `name = {{$json.full_name}}`
    - `email = {{$json.email}}`
    - `title, location, company_size, company_domain, company_industry`
    - `linkedin_profile_url = {{ $('On form submission1').item.json.Linkedin }}`
    - `company_revenue_range = {{$json.company_revenue}}`
    - `email_sent = false`
- **Connections:**
  - Output → `Your Offer`
- **Failure modes / edge cases:**
  - Data Table not created or schema mismatched (write fails).
  - Duplicate leads: no dedupe logic (matchingColumns is empty). Same LinkedIn can be inserted multiple times.
  - If `email` is missing, you still insert a row and later create a draft to an empty address unless you add validation.

---

### Block 3 — Prospect & Company Research (Perplexity + GPT Agent)

**Overview:** Uses an AI Agent configured as a “Lead Research Agent” with Perplexity as a tool and GPT-5-mini as the model to produce a 200–300 word research summary emphasizing the SDR/BDR hiring signal plus recent growth/GTM signals.

**Nodes involved:**
- `Your Offer` (feeds context into the agent prompt)
- `Lead Research Agent`
- `OpenAI Chat Model2`
- `Perplexity Research`
- `Simple Memory`

#### Node: Your Offer
- **Type / role:** `Set` — stores reusable business context for prompting.
- **Config (interpreted):** Defines long-form strings:
  - **Grand Slam Offer**
  - **Business Profile**
  - **Case Study** (note: this field is not directly used by the email agent; the agent instead calls the “Case Studies” tool)
- **Connections:**
  - Output → `Lead Research Agent`
- **Failure modes / edge cases:**
  - Overly long context can increase token usage and cost.
  - Claims like “replaces $10,500+/month SDRs” may create tone/legal/compliance risk depending on your market.

#### Node: Lead Research Agent
- **Type / role:** `LangChain Agent` — composes a research report using:
  - GPT model (`OpenAI Chat Model2`)
  - Web research tool (`Perplexity Research`)
  - Conversation memory (`Simple Memory`)
- **Config (interpreted):**
  - Prompt instructs research categories (growth signals, GTM context, scaling challenges, etc.)
  - System message injects lead/company fields from `Lead Information`
  - Reinforces the “primary intent signal” = hiring SDR/BDRs
- **Tools / attachments:**
  - Perplexity is connected as `ai_tool`
  - Memory buffer is connected as `ai_memory`
  - Model is connected as `ai_languageModel`
- **Outputs:**
  - Downstream references use: `{{ $('Lead Research Agent').item.json.output }}`
- **Connections:**
  - Main output → `Ice Breaker Email Generator`
- **Failure modes / edge cases:**
  - Perplexity tool errors (rate limits, network, missing citations).
  - Hallucination risk if research is not verifiable; prompt asks for sources but no strict citation schema is enforced.
  - Memory uses a fixed `sessionKey` (see `Simple Memory`) which can leak context across runs.

#### Node: OpenAI Chat Model2
- **Type / role:** `OpenAI Chat Model` — language model backing the research agent.
- **Config (interpreted):**
  - Model: **gpt-5-mini-2025-08-07**
- **Credentials:** OpenAI API credential required.
- **Connections:**
  - Provides `ai_languageModel` → `Lead Research Agent`
- **Failure modes / edge cases:**
  - Model availability/renaming; if not present in your account/region the node fails.
  - Token limits if Perplexity returns large text.

#### Node: Perplexity Research
- **Type / role:** `Perplexity Tool` — web search/research callable by the agent.
- **Config (interpreted):**
  - The message content is filled by the agent via `$fromAI('message0_Text', ...)`
  - `simplify: true` (returns simplified output)
- **Credentials:** Perplexity API credential required.
- **Connections:**
  - Tool output → available to `Lead Research Agent` via `ai_tool`
- **Failure modes / edge cases:**
  - API quota/rate limits.
  - Tool may return incomplete or noisy results; agent prompt should handle uncertainty.

#### Node: Simple Memory
- **Type / role:** `Memory Buffer Window` — maintains short conversational state for the agent.
- **Config (interpreted):**
  - Session ID type: custom key
  - Session key: **1020293294**
- **Connections:**
  - Memory → `Lead Research Agent` (`ai_memory`)
- **Failure modes / edge cases:**
  - Using a constant session key means *all executions share the same memory*, which can contaminate research between different leads. For per-lead isolation, use the lead ID or LinkedIn URL as session key.

---

### Block 4 — Email Personalization (Case Studies + GPT-5 + Structured JSON)

**Overview:** Uses a second AI Agent to write the final cold email. It consumes the research summary, your business profile/offer, and can fetch case studies from a data table tool. Output is forced into structured JSON by the output parser.

**Nodes involved:**
- `Ice Breaker Email Generator`
- `OpenAI Chat Model`
- `Structured Output Parser`
- `Return Case Studies`

#### Node: Ice Breaker Email Generator
- **Type / role:** `LangChain Agent` — generates subject + HTML email body.
- **Config (interpreted):**
  - Instructions enforce:
    - Subject 4–6 words, punchy
    - Body <120 words
    - Must reference SDR/BDR hiring in first sentence
    - Second sentence must be a provocative question with cost/risk/urgency
    - Third sentence must contain proof/metric
    - HTML output uses only plain text + `<br>`
  - System message injects:
    - Business Profile and Offer from `Your Offer`
    - Lead/company attributes from `Lead Information`
    - Research report from `Lead Research Agent` via `{{ ...json.output }}`
  - Output parser enabled (`hasOutputParser: true`)
  - Tool: “Case Study Tool” is provided by `Return Case Studies`
- **Connections:**
  - Model (`ai_languageModel`) from `OpenAI Chat Model`
  - Output parser (`ai_outputParser`) from `Structured Output Parser`
  - Tool (`ai_tool`) from `Return Case Studies`
  - Main output → `Create a draft`
- **Failure modes / edge cases:**
  - Prompt requires output JSON with keys `{subject, html_body}`, but downstream Gmail node expects different keys (see Block 5). This is a critical integration mismatch.
  - If research output is empty or weak, agent may over-generalize; you may want fallback logic (Template C).
  - HTML restrictions can be violated by the model; parser may fail if JSON invalid.

#### Node: OpenAI Chat Model
- **Type / role:** `OpenAI Chat Model` — language model backing the email generator.
- **Config (interpreted):**
  - Model: **gpt-5**
- **Credentials:** OpenAI API credential required.
- **Connections:**
  - Provides `ai_languageModel` → `Ice Breaker Email Generator`
- **Failure modes / edge cases:**
  - Model availability.
  - Higher cost than mini model.

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser` — validates/parses model output to JSON.
- **Config (interpreted):**
  - Provides an example schema with keys:
    - `lead_full_name`
    - `subject_line`
    - `email_address`
    - `icebreaker_email`
- **Connections:**
  - Parser → `Ice Breaker Email Generator` (`ai_outputParser`)
- **Failure modes / edge cases:**
  - **Schema mismatch:** The email agent prompt says output must be:
    ```json
    { "subject": "...", "html_body": "..." }
    ```
    but the parser example suggests different keys (subject_line, icebreaker_email, etc.). Depending on node behavior, this may cause parse errors or unexpected output structure.

#### Node: Return Case Studies
- **Type / role:** `Data Table Tool` — tool callable by the email agent to fetch case studies.
- **Config (interpreted):**
  - Operation: **get**
  - Return all rows: **true**
  - Data Table: **Case Studies** (id `BI2roIv7n6xCdRP2`)
- **Connections:**
  - Tool → `Ice Breaker Email Generator` (`ai_tool`)
- **Failure modes / edge cases:**
  - Missing table / empty results.
  - If too many case studies, the tool returns large payloads increasing token usage; consider filtering.

---

### Block 5 — Email Staging (Gmail Draft)

**Overview:** Creates a Gmail draft using the AI-generated content.

**Nodes involved:**
- `Create a draft`

#### Node: Create a draft
- **Type / role:** `Gmail` — create a draft email (HTML).
- **Config (interpreted):**
  - Resource: **draft**
  - Email type: **html**
  - Subject: `={{ $json.output.subject_line }}`
  - Message body: `={{ $json.output.icebreaker_email }}`
  - Recipient: `={{ $json.output.email_address }}`
- **Credentials:** Gmail OAuth2 credential required.
- **Connections:** terminal
- **Failure modes / edge cases:**
  - **Key mismatch with email generator:** If `Ice Breaker Email Generator` outputs `{subject, html_body}`, this node will not find `subject_line/icebreaker_email/email_address` and will create invalid drafts or fail.
  - Gmail OAuth scopes/refresh token issues.
  - If recipient email is empty/invalid, draft creation can fail.

**Practical fix:** Align the email generator + output parser + Gmail fields to the same key names (either update the agent to output `subject_line/icebreaker_email/email_address` or update Gmail to use `subject/html_body` and pull recipient from `Lead Information.email`).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission1 | Form Trigger | Entry point; collect LinkedIn URL | — | Email Finder | ## **1️⃣ Enrich Leads & Verify Emails**<br>- **Wiza Integration:** Submit a LinkedIn URL and Wiza finds verified email, job title, and company data.<br>- **Smart Routing:** Failed enrichments are flagged automatically so you never send to bad addresses. |
| Email Finder | Wiza | Enrich lead + find verified email from LinkedIn URL | On form submission1 | Switch1 | ## **1️⃣ Enrich Leads & Verify Emails**<br>- **Wiza Integration:** Submit a LinkedIn URL and Wiza finds verified email, job title, and company data.<br>- **Smart Routing:** Failed enrichments are flagged automatically so you never send to bad addresses. |
| Switch1 | Switch | Route based on Wiza status (failed/finished) | Email Finder | Failed To Find Enrichment; Lead Information | ## **1️⃣ Enrich Leads & Verify Emails**<br>- **Wiza Integration:** Submit a LinkedIn URL and Wiza finds verified email, job title, and company data.<br>- **Smart Routing:** Failed enrichments are flagged automatically so you never send to bad addresses. |
| Failed To Find Enrichment | Form | Show completion message when enrichment fails | Switch1 (Failed) | — | ## **1️⃣ Enrich Leads & Verify Emails**<br>- **Wiza Integration:** Submit a LinkedIn URL and Wiza finds verified email, job title, and company data.<br>- **Smart Routing:** Failed enrichments are flagged automatically so you never send to bad addresses. |
| Lead Information | Set | Normalize Wiza fields into canonical lead schema | Switch1 (Finished) | Add To lead Database |  |
| Add To lead Database | Data Table | Persist lead record into “Leads” table | Lead Information | Your Offer |  |
| Your Offer | Set | Store business profile + offer text for prompting | Add To lead Database | Lead Research Agent | ## **2️⃣ Research Each Prospect with AI**<br>- **Deep Research:** Perplexity AI searches the web for recent news, funding signals, and growth indicators about each lead.<br>- **Auto-Context:** All research is passed directly to the email writer for hyper-personalization. |
| Simple Memory | Memory Buffer Window | Provide agent memory (currently shared session key) | — | Lead Research Agent (ai_memory) | ## **2️⃣ Research Each Prospect with AI**<br>- **Deep Research:** Perplexity AI searches the web for recent news, funding signals, and growth indicators about each lead.<br>- **Auto-Context:** All research is passed directly to the email writer for hyper-personalization. |
| OpenAI Chat Model2 | OpenAI Chat Model | LLM for research agent | — | Lead Research Agent (ai_languageModel) | ## **2️⃣ Research Each Prospect with AI**<br>- **Deep Research:** Perplexity AI searches the web for recent news, funding signals, and growth indicators about each lead.<br>- **Auto-Context:** All research is passed directly to the email writer for hyper-personalization. |
| Perplexity Research | Perplexity Tool | Web research tool callable by research agent | — (called by agent) | Lead Research Agent (ai_tool) | ## **2️⃣ Research Each Prospect with AI**<br>- **Deep Research:** Perplexity AI searches the web for recent news, funding signals, and growth indicators about each lead.<br>- **Auto-Context:** All research is passed directly to the email writer for hyper-personalization. |
| Lead Research Agent | LangChain Agent | Generate research report from lead/company context + Perplexity | Your Offer (+ ai_model/tool/memory) | Ice Breaker Email Generator | ## **2️⃣ Research Each Prospect with AI**<br>- **Deep Research:** Perplexity AI searches the web for recent news, funding signals, and growth indicators about each lead.<br>- **Auto-Context:** All research is passed directly to the email writer for hyper-personalization. |
| Return Case Studies | Data Table Tool | Provide case studies tool to email agent | — | Ice Breaker Email Generator (ai_tool) | ## **3️⃣ Generate & Stage Personalized Emails**<br>- **Custom Writing:** GPT writes a unique icebreaker email for every person using real research and your case studies.<br>- **Draft Ready:** All personalized emails are pushed to Gmail as drafts, ready for review and send. |
| OpenAI Chat Model | OpenAI Chat Model | LLM for email generation | — | Ice Breaker Email Generator (ai_languageModel) | ## **3️⃣ Generate & Stage Personalized Emails**<br>- **Custom Writing:** GPT writes a unique icebreaker email for every person using real research and your case studies.<br>- **Draft Ready:** All personalized emails are pushed to Gmail as drafts, ready for review and send. |
| Structured Output Parser | Structured Output Parser | Enforce structured JSON output from email agent | — | Ice Breaker Email Generator (ai_outputParser) | ## **3️⃣ Generate & Stage Personalized Emails**<br>- **Custom Writing:** GPT writes a unique icebreaker email for every person using real research and your case studies.<br>- **Draft Ready:** All personalized emails are pushed to Gmail as drafts, ready for review and send. |
| Ice Breaker Email Generator | LangChain Agent | Produce subject + HTML body using research + offer + case studies | Lead Research Agent (+ ai_model/tool/parser) | Create a draft | ## **3️⃣ Generate & Stage Personalized Emails**<br>- **Custom Writing:** GPT writes a unique icebreaker email for every person using real research and your case studies.<br>- **Draft Ready:** All personalized emails are pushed to Gmail as drafts, ready for review and send. |
| Create a draft | Gmail | Create Gmail draft to enriched recipient | Ice Breaker Email Generator | — | ## **3️⃣ Generate & Stage Personalized Emails**<br>- **Custom Writing:** GPT writes a unique icebreaker email for every person using real research and your case studies.<br>- **Draft Ready:** All personalized emails are pushed to Gmail as drafts, ready for review and send. |
| Sticky Note | Sticky Note | Comment block label | — | — |  |
| Sticky Note1 | Sticky Note | Workflow overview + link | — | — |  |
| Sticky Note2 | Sticky Note | Comment block label | — | — |  |
| Sticky Note3 | Sticky Note | Comment block label | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a Form Trigger**
   - Node: **Form Trigger**
   - Title: “Linkedin”
   - Field: `Linkedin` (text)
   - Save and note the form URL (for submissions).

2) **Add Wiza enrichment**
   - Node: **Wiza** (Email Finder)
   - Input type: `linkedinUrl`
   - LinkedIn URL expression: `{{$json.Linkedin}}`
   - Add **Wiza API credentials**.

3) **Add routing on Wiza status**
   - Node: **Switch**
   - Rule 1 (rename output “Failed”): `{{$json.status}} equals "failed"`
   - Rule 2 (rename output “Finished”): `{{$json.status}} equals "finished"`
   - Connect: Form Trigger → Wiza → Switch

4) **Add failure completion page**
   - Node: **Form** (operation: completion)
   - Title/message as in workflow
   - Connect: Switch (Failed) → this node

5) **Normalize lead fields**
   - Node: **Set** (“Lead Information”)
   - Map from Wiza output to:
     - full_name, title, location, company_industry, company_domain, company_revenue (from company_revenue_range), company_size, company_type, company_description, company_location, email, id
   - Connect: Switch (Finished) → Lead Information

6) **Create “Leads” Data Table**
   - In n8n: create a Data Table named **Leads**
   - Include columns used by the workflow (name, linkedin_profile_url, email, title, location, company_size, company_industry, company_revenue_range, company_domain, email_sent, etc.)

7) **Insert lead into Leads table**
   - Node: **Data Table** (“Add To lead Database”)
   - Operation: create/insert row
   - Map:
     - `linkedin_profile_url = {{ $('On form submission1').item.json.Linkedin }}`
     - `email_sent = false`
     - other fields from `Lead Information`
   - Connect: Lead Information → Add To lead Database

8) **Add “Your Offer” context node**
   - Node: **Set** (“Your Offer”)
   - Add long text fields:
     - Business Profile
     - Grand Slam Offer
     - (optional) Case Study
   - Connect: Add To lead Database → Your Offer

9) **Add Perplexity tool node**
   - Node: **Perplexity Tool** (“Perplexity Research”)
   - Keep default tool message input using `$fromAI(...)`
   - Add **Perplexity API credentials**

10) **Add OpenAI model for research**
   - Node: **OpenAI Chat Model** (“OpenAI Chat Model2”)
   - Model: `gpt-5-mini-2025-08-07`
   - Add **OpenAI API credentials**

11) **Add Memory (recommended: make it per-lead)**
   - Node: **Memory Buffer Window**
   - Instead of a constant session key, use an expression like:
     - Session key: `{{ $('Lead Information').item.json.email || $('On form submission1').item.json.Linkedin }}`
   - (Current workflow uses a fixed key; changing it prevents cross-lead contamination.)

12) **Create the research agent**
   - Node: **LangChain Agent** (“Lead Research Agent”)
   - Prompt: research instructions (growth signals, GTM, etc.) and emphasize SDR/BDR hiring signal
   - System message: inject lead/company fields and instruct verifiable facts
   - Attach:
     - `OpenAI Chat Model2` to agent as **ai_languageModel**
     - `Perplexity Research` to agent as **ai_tool**
     - `Simple Memory` to agent as **ai_memory**
   - Connect: Your Offer → Lead Research Agent

13) **Create “Case Studies” Data Table**
   - Create a Data Table named **Case Studies**
   - Add rows with short case study blurbs and any tags/industry fields you want for filtering later.

14) **Add Case Studies tool**
   - Node: **Data Table Tool** (“Return Case Studies”)
   - Operation: `get`, Return All: true
   - Select Data Table: **Case Studies**

15) **Add OpenAI model for email writing**
   - Node: **OpenAI Chat Model** (“OpenAI Chat Model”)
   - Model: `gpt-5`
   - Use same OpenAI credentials (or separate).

16) **Add structured output parser**
   - Node: **Structured Output Parser**
   - Ensure the schema keys match what you will use downstream (see next step).

17) **Create the email generator agent**
   - Node: **LangChain Agent** (“Ice Breaker Email Generator”)
   - Prompt: your style rules + templates, enforce JSON output
   - System message: inject
     - Business Profile / Offer from `Your Offer`
     - Lead Information fields
     - Research output from `Lead Research Agent`
   - Attach:
     - `OpenAI Chat Model` as **ai_languageModel**
     - `Return Case Studies` as **ai_tool**
     - `Structured Output Parser` as **ai_outputParser**
   - Connect: Lead Research Agent → Ice Breaker Email Generator

18) **Create Gmail draft**
   - Node: **Gmail** (“Create a draft”)
   - Resource: `draft`
   - Email type: `html`
   - Add **Gmail OAuth2 credentials**
   - Map fields **consistently** with your agent output:
     - If agent outputs `{ "subject": "...", "html_body": "..." }` then set:
       - Subject: `{{$json.output.subject}}`
       - Message: `{{$json.output.html_body}}`
       - To: `{{ $('Lead Information').item.json.email }}`
     - If agent outputs `{ "subject_line": "...", "icebreaker_email": "...", "email_address": "..." }` then keep current mappings.
   - Connect: Ice Breaker Email Generator → Create a draft

19) **Activate workflow**
   - Submit a LinkedIn URL via the form and confirm:
     - Wiza returns finished
     - Lead is inserted in Leads table
     - Research output exists
     - Draft appears in Gmail

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “A 100% automated system that enriches LinkedIn leads, researches their companies with AI, and writes personalized cold emails that actually get replies.” | Workflow overview sticky note |
| APIs required: Wiza, OpenAI, Perplexity, Gmail | Setup requirements |
| “Find all downloadable resources including detailed set up information and database template csv in the Full Notion Documentation” | https://freemezie.notion.site/Email-Outreach-Automation-2d4c8a2a63e080dfa4e8e0fe6ea27894?source=copy_link |
| Setup steps noted: add credentials, create “Leads” & “Case Studies” tables, update “Your Offer”, activate workflow and use form URL | Operational guidance |

Disclaimer (provided by you): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…*