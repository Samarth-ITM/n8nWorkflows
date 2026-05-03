Scrape LinkedIn B2B leads with Apify and GPT-4 and approve emails in Sheets

https://n8nworkflows.xyz/workflows/scrape-linkedin-b2b-leads-with-apify-and-gpt-4-and-approve-emails-in-sheets-13007


# Scrape LinkedIn B2B leads with Apify and GPT-4 and approve emails in Sheets

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Scrape LinkedIn B2B leads with Apify and GPT-4 and approve emails in Sheets  
**Purpose:** End-to-end B2B lead pipeline: collect search keywords, scrape leads via Apify, filter for leads with emails, create CRM records in HubSpot, generate personalized cold emails with an OpenAI chat model, log everything to Google Sheets with **Pending** status, and support a human-in-the-loop review where **Approve** sends the email via Gmail and **Reject** rewrites the email and updates the sheet.

### 1.1 Campaign Input & Keyword Expansion
Collect lead targeting keywords from an n8n form and split them into individual keywords for iterative processing.

### 1.2 Lead Scraping Orchestration (Apify)
Trigger an Apify Actor run (LinkedIn leads scraping) and then fetch the dataset items from the last run.

### 1.3 Lead Iteration & Email Existence Filtering
Batch-process scraped leads and keep only items that contain an email address.

### 1.4 CRM Sync (HubSpot)
Create/ensure a company exists, then create/update a contact associated with that company.

### 1.5 AI Email Draft Generation (OpenAI via LangChain Agent)
Convert batched lead JSON into a single JSON string, feed it to an agent with a strict system prompt, and produce one cold email per lead.

### 1.6 Logging & Human Review in Google Sheets
Append/update lead rows (matched by email) and store generated email drafts with **Pending** status.

### 1.7 Approval Path: Send Email (Webhook → Gmail)
When a lead is approved externally (via webhook), normalize payload, parse the AI email into subject/body, and send via Gmail.

### 1.8 Rejection Path: Rewrite Email (Webhook → OpenAI → Sheets Update)
When rejected (via webhook), normalize payload, rewrite email with a different angle using the chat model, and update the same sheet row (matched by email).

---

## 2. Block-by-Block Analysis

### Block 1 — Campaign Input & Keyword Expansion

**Overview:** Collects a keyword string from a form and splits it into multiple items (one keyword per item) so downstream nodes can scrape per keyword.  
**Nodes involved:** `Lead Campaign Setup`, `make separate json of each keyword`

#### Node: Lead Campaign Setup
- **Type / role:** Form Trigger (`n8n-nodes-base.formTrigger`) — entry point to start a campaign.
- **Configuration choices:**
  - Form title: “LinkedIn Leads Scraping Setup”
  - One required field: **Keyword**
  - Response via built-in form mechanism (webhookId present).
- **Key variables/fields:** Produces JSON containing `Keyword`.
- **Connections:** Outputs to `make separate json of each keyword`.
- **Edge cases / failures:**
  - Empty/poorly formatted keyword string leads to no usable keywords downstream (though the form enforces required field).
  - If users separate keywords with single spaces, the current splitter may not split as expected (see next node).

#### Node: make separate json of each keyword
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms a single string into multiple n8n items.
- **Configuration choices (logic):**
  - Reads `input.Keyword`.
  - Splits by **2+ spaces** regex: `/\s{2,}/`
  - Trims and filters empties.
  - Returns items as: `{ json: { keyword: <k> } }`
- **Connections:** Outputs to `Data Batcher`.
- **Edge cases / failures:**
  - If keywords are separated by commas, newlines, or single spaces, they won’t split as intended. Users must use **double spaces** (or you should adjust regex).
  - If `Keyword` missing, defaults to `""`, resulting in zero items.

---

### Block 2 — Lead Scraping Orchestration (Apify)

**Overview:** For each keyword batch iteration, calls Apify Actor run endpoint, then retrieves dataset items from the latest run.  
**Nodes involved:** `Data Batcher`, `post Apify data scrap`, `Get Apify recent run data`

#### Node: Data Batcher
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — controls iteration over keyword items.
- **Configuration choices:**
  - Default options (batch size default depends on n8n UI; not explicitly set here).
  - Uses the **second output** to loop: its output #1 is connected to scraping.
- **Connections:**
  - Input: `make separate json of each keyword`
  - Output(1): (unused)
  - Output(2): `post Apify data scrap`
- **Edge cases / failures:**
  - If batch size is >1, you may be triggering Apify with multiple keywords without using them (see next node; keyword isn’t referenced at all in request body).
  - Looping behavior depends on having downstream complete; mismatched usage can cause partial processing.

#### Node: post Apify data scrap
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — starts an Apify Actor run.
- **Configuration choices:**
  - POST to: `https://api.apify.com/v2/acts/peakydev~leads-scraper-ppe/runs?token=YOUR_TOKEN_HERE`
  - Auth: **HTTP Header Auth** credential named `Aliapify`
  - Sends JSON body:
    - `includeEmails: true`
    - `industry: ["Primary and Secondary Education"]`
    - `totalResults: 10000`
  - Header: `Accept: Application/json` (note capitalization; typically `application/json`).
- **Notable limitation:** The request body does **not** use the per-item `keyword` produced earlier, so changing keywords currently has no effect unless the Apify actor infers something else.
- **Connections:** Outputs to `Get Apify recent run data`.
- **Edge cases / failures:**
  - Token placeholder not replaced → 401/403.
  - Actor run may be asynchronous; immediately fetching “last dataset items” can race and return incomplete data.
  - Very high `totalResults` can cause long runs, rate limits, or cost spikes.

#### Node: Get Apify recent run data
- **Type / role:** HTTP Request — fetches dataset items from the last run.
- **Configuration choices:**
  - GET: `https://api.apify.com/v2/acts/peakydev~leads-scraper-ppe/runs/last/dataset/items?token=YOUR_TOKEN_HERE`
  - Same `Aliapify` header auth and Accept header.
- **Connections:** Outputs to `Lead Data Batcher`.
- **Edge cases / failures:**
  - “runs/last” might point to a run not initiated by this workflow if multiple runs exist.
  - If the run is not finished, dataset may be empty/partial.
  - Large datasets can exceed n8n memory/time limits.

---

### Block 3 — Lead Iteration & Email Existence Filtering

**Overview:** Iterates over scraped lead items in batches and filters out items without an email field.  
**Nodes involved:** `Lead Data Batcher`, `If email exist`

#### Node: Lead Data Batcher
- **Type / role:** Split In Batches — processes scraped leads incrementally.
- **Configuration choices:** Default options.
- **Connections:**
  - Input: `Get Apify recent run data`
  - Output(1): loops back to `Data Batcher` (this is unusual; it links lead-batching back to keyword batching).
  - Output(2): `If email exist`
- **Edge cases / failures:**
  - The connection from this node back to `Data Batcher` can create unexpected looping semantics (keywords vs leads). This can cause:
    - Re-triggering scraping while still processing leads.
    - Hard-to-debug execution graphs and possible infinite loops if not carefully controlled by batch completion.
  - Consider separating keyword iteration from lead iteration.

#### Node: If email exist
- **Type / role:** IF (`n8n-nodes-base.if`) — keeps only leads with an `email` field.
- **Configuration choices:**
  - Condition: `={{ $json.email }}` **exists**
  - Strict type validation enabled.
- **Connections:**
  - True output → `JSON Stringifier`
  - False output → `Lead Data Batcher` (i.e., skip lead and continue batching)
- **Edge cases / failures:**
  - If email field is nested or named differently (`emails`, `contact.email`, etc.), valid leads may be dropped.
  - “exists” does not validate format; invalid emails still pass.

---

### Block 4 — CRM Sync (HubSpot)

**Overview:** Creates a company record then syncs a contact and associates it to the company.  
**Nodes involved:** `JSON Stringifier`, `HubSpot Company Creator`, `HubSpot Contact Sync`

#### Node: JSON Stringifier
- **Type / role:** Code — collects incoming items and serializes them into a JSON string under `userInput`.
- **Configuration choices (logic):**
  - `const items = $input.all()`
  - `userInput = JSON.stringify(items.map(item => item.json))`
- **Connections:** Output → `HubSpot Company Creator`
- **Edge cases / failures:**
  - If the upstream is item-by-item (not aggregated), `$input.all()` may contain more than intended, impacting prompt size later (though in this workflow it’s also referenced later by the AI agent in a non-standard way).
  - Large lead objects can exceed memory and downstream model token limits.

#### Node: HubSpot Company Creator
- **Type / role:** HubSpot (`n8n-nodes-base.hubspot`) — creates a company.
- **Configuration choices:**
  - Authentication: **App Token**
  - Resource: `company`
  - Fields mapped from `If email exist` item (using `$item("0").$node["If email exist"].json[...]`):
    - `name` ← organizationName
    - `websiteUrl` ← organizationWebsite
    - `description` ← organizationDescription
    - `yearFounded` ← organizationFoundedYear
    - `linkedInCompanyPage` ← organizationLinkedinUrl
- **Connections:** Output → `HubSpot Contact Sync`
- **Edge cases / failures:**
  - Missing fields cause empty values; HubSpot may reject invalid `yearFounded`.
  - Duplicate companies: unless HubSpot node is configured to upsert/search (not shown), it may create duplicates.
  - Expression references assume item index `0`; if multi-item context differs, mapping can break.

#### Node: HubSpot Contact Sync
- **Type / role:** HubSpot — creates/updates a contact and associates to company.
- **Configuration choices:**
  - Uses `email` as primary identifier.
  - Additional fields mapped from `If email exist`:
    - city, country, stateRegion, jobTitle, firstName, lastName, companySize, linkedinUrl
    - `associatedCompanyId` ← `={{ $json.companyId }}` (from previous node’s output)
- **Connections:** Output → `generate Ai email`
- **Edge cases / failures:**
  - If company creation fails, `companyId` absent and association fails.
  - If lead data lacks `firstName`/`lastName` fields but prompt expects `fullName`, you may have inconsistent schemas.
  - HubSpot rate limits for large batches.

---

### Block 5 — AI Email Draft Generation (OpenAI via LangChain)

**Overview:** Generates personalized outreach email text.  
**Nodes involved:** `LLM`, `generate Ai email`

#### Node: LLM
- **Type / role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the language model to the agent.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - Credentials: OpenAI API key (`Ali Hassan openAI`)
- **Connections:** Connected as `ai_languageModel` input to `generate Ai email`.
- **Edge cases / failures:**
  - Model availability/permissions.
  - Token limits if `userInput` is large.
  - Cost and latency considerations.

#### Node: generate Ai email
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — runs the prompt and produces email drafts.
- **Configuration choices:**
  - Prompt type: define
  - **System message:** detailed rules for generating one email per lead; prohibits mentioning LinkedIn/scraping; outputs plain text.
  - **Text input expression:**
    - `=User Input:  {{ $item("0").$node["JSON Stringifier"].json["userInput"] }}`
- **Connections:** Output → `Leads Log`
- **Edge cases / failures:**
  - The expression references `JSON Stringifier` output; ensure that node executed in the same run branch and item context is valid.
  - Output expected at `$json.output` downstream (Sheets nodes use that); if the agent returns a different field name, logging breaks.
  - If multiple leads are provided, prompt requests multiple emails; downstream sheet logging currently writes a single `emailContent` per row, which may mismatch.

---

### Block 6 — Logging & Human Review (Google Sheets)

**Overview:** Writes lead details and generated email drafts into Google Sheets with “Pending” status; uses email as matching key.  
**Nodes involved:** `Leads Log`

#### Node: Leads Log
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append or update lead rows.
- **Configuration choices:**
  - Operation: **appendOrUpdate**
  - Matching column: `email`
  - Sheet: Spreadsheet “Linkdin Leads Scrapping”, tab “Leads” (gid=0)
  - Writes columns including:
    - lead metadata from `If email exist` (city/state/country/org fields…)
    - `Status: "Pending"`
    - `emailContent: ={{ $json.output }}` (expects AI output in `output`)
- **Connections:** Output → `Lead Data Batcher` (continues processing)
- **Edge cases / failures:**
  - If `$json.output` doesn’t exist (agent output schema differs), emailContent will be blank.
  - If email duplicates across leads, row will update instead of append (by design).
  - Google Sheets API quotas; authentication expiry.

---

### Block 7 — Approved Leads Email Workflow (Webhook → Gmail)

**Overview:** Receives an approval event (likely from a Sheets UI/button automation), normalizes fields, extracts subject/body from stored email content, and sends the email through Gmail.  
**Nodes involved:** `📥 Approved Leads Webhook`, `🔧 Normalize Lead Data`, `✂️ Split Email Content`, `📤 Send Approved Email`

#### Node: 📥 Approved Leads Webhook
- **Type / role:** Webhook (`n8n-nodes-base.webhook`) — entry point for approvals.
- **Configuration choices:**
  - Path: `webhook/approved`
  - Method: POST
  - Response: last node
- **Connections:** Output → `🔧 Normalize Lead Data`
- **Edge cases / failures:**
  - Must be publicly reachable or invoked internally with correct URL.
  - Caller must send `body` with expected fields (see normalizer).

#### Node: 🔧 Normalize Lead Data
- **Type / role:** Code — maps webhook body into a normalized schema aligned with Sheets columns and email sending.
- **Configuration choices (logic):**
  - Reads `webhookData.body` as `data`
  - Produces normalized fields (organizationIndustry, email, location, org fields, emailContent, Status, fullName/position/organizationName, tracking fields)
- **Connections:** Output → `✂️ Split Email Content`
- **Edge cases / failures:**
  - If webhook payload doesn’t wrap data inside `body`, fields will be blank.
  - Logs to console (useful in dev, noisy in prod).

#### Node: ✂️ Split Email Content
- **Type / role:** Code — parses “Subject:” line and body from the emailContent text.
- **Configuration choices (logic):**
  - Attempts multiple paths to locate `emailContent`
  - Regex extracts subject: `/Subject:\s*(.+?)(?:\n\n|\r\n\r\n|$)/im`
  - Defaults subject: `"Automation Proposal"`
- **Connections:** Output → `📤 Send Approved Email`
- **Edge cases / failures:**
  - If AI output doesn’t include `Subject:` line, subject defaults and body may include full text.
  - If content contains escaped newlines, it tries to unescape.
  - If email content includes multiple emails, this node will treat it as one message.

#### Node: 📤 Send Approved Email
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends the approved email.
- **Configuration choices:**
  - To: hardcoded `muhammadmoosa.abc1@gmail.com` (not dynamic per lead)
  - Subject: `={{ $json.subject }}`
  - Message: `={{ $json.body }}`
  - Email type: text
- **Connections:** Terminal node.
- **Edge cases / failures:**
  - The recipient is not derived from lead data; if the intent is to email the lead, this must be changed to `={{ $node["🔧 Normalize Lead Data"].json.email }}` or equivalent.
  - Gmail OAuth scopes and sending limits.

---

### Block 8 — Rejection Follow-up Workflow (Webhook → Rewrite → Sheets Update)

**Overview:** When an email draft is rejected, the workflow rewrites it with a new angle and updates the Google Sheet row matched by email.  
**Nodes involved:** `Rejection Webhook`, `Webhook Data Normalizer`, `Rejection Data Stringifier`, `OpenAI Chat Model`, `Rejection Email Rewriter`, `Update: Improved Email`

#### Node: Rejection Webhook
- **Type / role:** Webhook — entry point for rejections.
- **Configuration choices:**
  - Path: `webhook/rejected`
  - Method: POST
  - Response: last node
- **Connections:** Output → `Webhook Data Normalizer`
- **Edge cases / failures:** Same as approval webhook (payload shape must match).

#### Node: Webhook Data Normalizer
- **Type / role:** Code — same normalization as approval path.
- **Configuration choices:** Maps `webhookData.body` into normalized fields, including `emailContent`, `Status`, and tracking.
- **Connections:** Output → `Rejection Data Stringifier`
- **Edge cases / failures:** Same as other normalizer.

#### Node: Rejection Data Stringifier
- **Type / role:** Code — converts incoming items to a JSON string `userInput`.
- **Configuration choices:** Same pattern as `JSON Stringifier`.
- **Connections:** Output → `Rejection Email Rewriter`
- **Edge cases / failures:** Large payloads can exceed model limits.

#### Node: OpenAI Chat Model
- **Type / role:** OpenAI chat model provider for the rejection agent.
- **Configuration choices:** Model `gpt-5-mini`.
- **Connections:** `ai_languageModel` → `Rejection Email Rewriter`.
- **Edge cases / failures:** Same as `LLM`.

#### Node: Rejection Email Rewriter
- **Type / role:** LangChain Agent — rewrites the rejected email.
- **Configuration choices:**
  - Input text: `=User input:  {{ $json.userInput }}`
  - System message: rewrite with new angle; no mention of rejection; soft CTA; output only email text.
- **Connections:** Output → `Update: Improved Email`
- **Edge cases / failures:**
  - If emailContent is absent, agent will create a new email but with limited context.
  - Downstream expects `$json.output` again.

#### Node: Update: Improved Email
- **Type / role:** Google Sheets — updates the emailContent for the row matching the email.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `email`
  - Writes:
    - `email` from normalizer
    - `emailContent` from `={{ $json.output }}`
- **Connections:** Terminal for rejection path.
- **Edge cases / failures:**
  - If multiple rows share an email, update target may be ambiguous.
  - If agent output field name differs, update writes blank.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Lead Campaign Setup | Form Trigger | Collect keyword criteria from user | — | make separate json of each keyword | # LinkedIn Lead Generation Pipeline |
| make separate json of each keyword | Code | Split keyword string into individual keyword items | Lead Campaign Setup | Data Batcher | # LinkedIn Lead Generation Pipeline |
| Data Batcher | Split In Batches | Iterate over keyword items (batch control) | make separate json of each keyword | post Apify data scrap | # LinkedIn Lead Generation Pipeline |
| post Apify data scrap | HTTP Request | Start Apify Actor run | Data Batcher | Get Apify recent run data | # LinkedIn Lead Generation Pipeline |
| Get Apify recent run data | HTTP Request | Fetch dataset items from last Apify run | post Apify data scrap | Lead Data Batcher | # LinkedIn Lead Generation Pipeline |
| Lead Data Batcher | Split In Batches | Iterate over scraped leads | Get Apify recent run data; Leads Log; If email exist (false path) | Data Batcher; If email exist | # LinkedIn Lead Generation Pipeline |
| If email exist | IF | Filter leads with existing email | Lead Data Batcher | JSON Stringifier (true); Lead Data Batcher (false) | # LinkedIn Lead Generation Pipeline |
| JSON Stringifier | Code | Serialize lead JSON list into `userInput` string | If email exist (true) | HubSpot Company Creator | # LinkedIn Lead Generation Pipeline |
| HubSpot Company Creator | HubSpot | Create company in HubSpot | JSON Stringifier | HubSpot Contact Sync | # LinkedIn Lead Generation Pipeline |
| HubSpot Contact Sync | HubSpot | Create/update contact and associate to company | HubSpot Company Creator | generate Ai email | # LinkedIn Lead Generation Pipeline |
| LLM | OpenAI Chat Model (LangChain) | LLM provider for email generation agent | — | generate Ai email (ai_languageModel) | # LinkedIn Lead Generation Pipeline |
| generate Ai email | LangChain Agent | Generate personalized cold email(s) | HubSpot Contact Sync | Leads Log | # LinkedIn Lead Generation Pipeline |
| Leads Log | Google Sheets | Append/update lead row + draft email, set Pending | generate Ai email | Lead Data Batcher | # LinkedIn Lead Generation Pipeline |
| 📥 Approved Leads Webhook | Webhook | Entry point when lead is approved | — | 🔧 Normalize Lead Data | # Approved Leads Email Workflow |
| 🔧 Normalize Lead Data | Code | Normalize approved payload to standard schema | 📥 Approved Leads Webhook | ✂️ Split Email Content | # Approved Leads Email Workflow |
| ✂️ Split Email Content | Code | Parse Subject and Body from emailContent | 🔧 Normalize Lead Data | 📤 Send Approved Email | # Approved Leads Email Workflow |
| 📤 Send Approved Email | Gmail | Send email via Gmail | ✂️ Split Email Content | — | # Approved Leads Email Workflow |
| Rejection Webhook | Webhook | Entry point when lead is rejected | — | Webhook Data Normalizer | # Rejection Follow-up Workflow |
| Webhook Data Normalizer | Code | Normalize rejected payload to standard schema | Rejection Webhook | Rejection Data Stringifier | # Rejection Follow-up Workflow |
| Rejection Data Stringifier | Code | Serialize normalized rejection input into `userInput` | Webhook Data Normalizer | Rejection Email Rewriter | # Rejection Follow-up Workflow |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for rejection rewrite agent | — | Rejection Email Rewriter (ai_languageModel) | # Rejection Follow-up Workflow |
| Rejection Email Rewriter | LangChain Agent | Rewrite rejected email with new angle | Rejection Data Stringifier | Update: Improved Email | # Rejection Follow-up Workflow |
| Update: Improved Email | Google Sheets | Update emailContent for matching email row | Rejection Email Rewriter | — | # Rejection Follow-up Workflow |
| Sticky Note | Sticky Note | Documentation / grouping | — | — |  |
| Sticky Note1 | Sticky Note | How it works + setup steps | — | — |  |
| Sticky Note4 | Sticky Note | Rejection section header | — | — |  |
| Sticky Note5 | Sticky Note | Approval section header | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   “Scrape LinkedIn B2B leads with Apify and GPT-4 and approve emails in Sheets”.

2. **Add Form Trigger** node:
   - Node name: `Lead Campaign Setup`
   - Title: “LinkedIn Leads Scraping Setup”
   - Field: `Keyword` (required)
   - Connect it to the next node.

3. **Add Code node**:
   - Node name: `make separate json of each keyword`
   - Paste logic to split `Keyword` by **2+ spaces**, returning items `{ keyword: ... }`.
   - Connect to `Data Batcher`.

4. **Add Split In Batches** node:
   - Node name: `Data Batcher`
   - Configure batch size as desired (e.g., 1 per keyword).
   - Connect its **loop/next output** (second output) to the Apify POST node.

5. **Add HTTP Request** node:
   - Node name: `post Apify data scrap`
   - Method: POST
   - URL: `https://api.apify.com/v2/acts/peakydev~leads-scraper-ppe/runs?token=YOUR_TOKEN_HERE`
   - Auth: HTTP Header Auth credential (create `Aliapify`)
     - Put Apify token in header per your auth scheme, and replace `YOUR_TOKEN_HERE` in URL or remove it and use headers only (recommended).
   - Body: JSON with your actor input (includeEmails, filters, etc.).
   - Connect to `Get Apify recent run data`.

6. **Add HTTP Request** node:
   - Node name: `Get Apify recent run data`
   - Method: GET
   - URL: `https://api.apify.com/v2/acts/peakydev~leads-scraper-ppe/runs/last/dataset/items?token=YOUR_TOKEN_HERE`
   - Same Apify credential.
   - Connect to `Lead Data Batcher`.

7. **Add Split In Batches** node:
   - Node name: `Lead Data Batcher`
   - Batch size: pick based on HubSpot/Sheets rate limits (e.g., 10–50).
   - Connect output to an IF node that checks email existence.
   - (Optional but recommended) avoid connecting back to `Data Batcher`; instead, keep keyword loop independent.

8. **Add IF node**:
   - Node name: `If email exist`
   - Condition: String → `{{ $json.email }}` → **exists**
   - True output → `JSON Stringifier`
   - False output → back to `Lead Data Batcher` to continue.

9. **Add Code node**:
   - Node name: `JSON Stringifier`
   - Serialize `$input.all()` into `{ userInput: "<json string>" }`.
   - Connect to HubSpot company node.

10. **Add HubSpot node (Company Create)**:
    - Node name: `HubSpot Company Creator`
    - Auth: HubSpot **App Token** credential
    - Resource: Company
    - Map fields from the lead JSON (organizationName, website, description, etc.).
    - Connect to `HubSpot Contact Sync`.

11. **Add HubSpot node (Contact Create/Update)**:
    - Node name: `HubSpot Contact Sync`
    - Use `email` as identifier
    - Map contact fields
    - Set `associatedCompanyId` from the previous company create result.
    - Connect to `generate Ai email`.

12. **Add OpenAI Chat Model node (LangChain)**:
    - Node name: `LLM`
    - Model: `gpt-5-mini` (or your preferred)
    - Credentials: OpenAI API key.
    - Connect as **AI Language Model** to the agent node.

13. **Add LangChain Agent node**:
    - Node name: `generate Ai email`
    - Prompt: define
    - System message: include the rules (no LinkedIn mention, plain text output, etc.)
    - Text input: pass `userInput` string (from `JSON Stringifier`).
    - Connect to `Leads Log`.

14. **Add Google Sheets node (Append/Update)**:
    - Node name: `Leads Log`
    - Auth: Google Sheets OAuth2 credential
    - Document: your spreadsheet
    - Sheet: “Leads”
    - Operation: appendOrUpdate
    - Matching column: `email`
    - Write lead fields + `Status = Pending` + `emailContent` from agent output.
    - Connect to `Lead Data Batcher` to continue processing.

15. **Approval path (add Webhook)**:
    - Node name: `📥 Approved Leads Webhook`
    - Path: `webhook/approved`
    - Method: POST
    - Connect to normalizer code node.

16. **Add Code node (approval normalizer)**:
    - Node name: `🔧 Normalize Lead Data`
    - Map `webhookData.body` to normalized structure and include `emailContent`.
    - Connect to email split node.

17. **Add Code node (split subject/body)**:
    - Node name: `✂️ Split Email Content`
    - Parse `Subject:` and remaining body.
    - Connect to Gmail send node.

18. **Add Gmail node (Send)**:
    - Node name: `📤 Send Approved Email`
    - Credential: Gmail OAuth2
    - Send To: set dynamic recipient (recommended) or keep fixed.
    - Subject/body from previous node.
    - This ends approval flow.

19. **Rejection path (add Webhook)**:
    - Node name: `Rejection Webhook`
    - Path: `webhook/rejected`
    - Method: POST
    - Connect to rejection normalizer.

20. **Add Code node (rejection normalizer)**:
    - Node name: `Webhook Data Normalizer`
    - Same mapping as approval.
    - Connect to `Rejection Data Stringifier`.

21. **Add Code node (rejection stringifier)**:
    - Node name: `Rejection Data Stringifier`
    - Build `userInput` string from incoming items.
    - Connect to `Rejection Email Rewriter`.

22. **Add OpenAI Chat Model node (LangChain)**:
    - Node name: `OpenAI Chat Model`
    - Model: `gpt-5-mini`
    - Connect as AI Language Model to `Rejection Email Rewriter`.

23. **Add LangChain Agent node (rewrite)**:
    - Node name: `Rejection Email Rewriter`
    - System message: instruct rewrite with different angle; output only email.
    - Input text: `userInput`.
    - Connect to Google Sheets update node.

24. **Add Google Sheets node (Update)**:
    - Node name: `Update: Improved Email`
    - Operation: Update
    - Matching column: `email`
    - Update `emailContent` with rewritten output.

**Credentials to configure:**
- Apify: HTTP Header Auth (or API token in URL, but header is better).
- OpenAI: API key for chat model nodes.
- HubSpot: App Token.
- Google Sheets: OAuth2.
- Gmail: OAuth2 with send scope.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “# LinkedIn Lead Generation Pipeline” | Sticky note section label for the main scraping → CRM → AI → Sheets flow |
| “# Approved Leads Email Workflow” | Sticky note section label for approval webhook → Gmail sending |
| “# Rejection Follow-up Workflow” | Sticky note section label for rejection webhook → rewrite → Sheets update |
| How it works + setup steps (full text preserved) | Sticky note content: explains the 8-step process and required credentials; includes reminders to update Sheet ID and AI signature text |