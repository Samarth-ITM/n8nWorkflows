Send AI sales proposals and Stripe payment links after Calendly calls

https://n8nworkflows.xyz/workflows/send-ai-sales-proposals-and-stripe-payment-links-after-calendly-calls-12530


# Send AI sales proposals and Stripe payment links after Calendly calls

## 1. Workflow Overview

**Purpose:**  
Automate the post–sales-call close process: when a meeting is booked in Calendly, the workflow pulls lead context from a Google Sheets “CRM”, generates a structured AI proposal, creates a Google Slides proposal deck from a template, creates a Stripe Checkout payment link, and emails the prospect a follow-up containing both links.

**Primary use cases:**
- Agencies/consultants sending proposals immediately after a discovery call
- Standardized “call → proposal → deposit” pipeline with minimal manual work
- Fast follow-up to reduce drop-off after meetings

### 1.1 Trigger & Lead Context
Starts from a Calendly booking event and enriches it with CRM data from Google Sheets.

### 1.2 AI Proposal Generation
Uses OpenAI (via the LangChain OpenAI node) to generate **strict JSON** proposal content from CRM fields.

### 1.3 Document Creation (Slides)
Copies a Google Slides template (via Drive copy) and injects AI-generated content into placeholders via Google Slides text replacement.

### 1.4 Payment & Follow-up
Creates a Stripe Checkout Session (Payment Link) and sends a personalized Gmail follow-up including the proposal deck URL and Stripe checkout URL.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Lead Context

**Overview (role):**  
Receives a “meeting booked” event from Calendly and uses the invitee email to look up the matching lead row in Google Sheets.

**Nodes involved:**
- Calendly Trigger Booked Meeting
- Find Client Details From CRM

#### Node: Calendly Trigger Booked Meeting
- **Type / role:** `Calendly Trigger` — webhook trigger for Calendly events.
- **Configuration (interpreted):**
  - Listens to event: **`invitee.created`** (a new invitee scheduled).
  - Auth: **OAuth2** (Calendly OAuth2 credential).
  - Produces payload including invitee email (`$json.payload.email`) and scheduled event info.
- **Key data used downstream:**
  - `{{$json.payload.email}}` is used as the CRM lookup key.
- **Connections:**
  - Output → **Find Client Details From CRM**
- **Version-specific notes:** Node typeVersion `1`. OAuth2 must be configured in n8n with Calendly.
- **Edge cases / failures:**
  - Webhook not registered/disabled if workflow inactive.
  - Calendly OAuth token expired or missing scopes.
  - Payload structure changes (Calendly API changes) could break `payload.email` reference.

#### Node: Find Client Details From CRM
- **Type / role:** `Google Sheets` — lookup/enrichment from a spreadsheet.
- **Configuration (interpreted):**
  - Spreadsheet: “CRM” (Google Sheet document).
  - Sheet/tab: `Sheet1` (gid=0).
  - Filter/lookup:
    - **Lookup column:** `Email`
    - **Lookup value:** `={{ $json.payload.email }}`
  - Returns the matched row as JSON (including fields like `Problem`, `Solution`, `Scope`, `Cost`, `Company Legal Name`, `Person Name`, etc.).
- **Key expressions / variables:**
  - `={{ $json.payload.email }}` (from Calendly trigger output)
- **Connections:**
  - Input ← **Calendly Trigger Booked Meeting**
  - Output → **Generate Proposal Copy1**
- **Version-specific notes:** typeVersion `4.7` (Sheets node UI/behavior can vary across versions).
- **Edge cases / failures:**
  - No matching row (downstream nodes may reference missing fields and error or produce poor prompts).
  - Duplicate emails (may return first match; verify expected behavior in your n8n version).
  - Google Sheets API quota/rate limiting.
  - OAuth permissions missing for the spreadsheet.

---

### Block 2 — Proposal Generation & Document Creation

**Overview (role):**  
Generates structured proposal content as JSON via OpenAI, copies a Slides template, then replaces placeholder tokens in the copied deck with AI output.

**Nodes involved:**
- Generate Proposal Copy1
- Create Proposal Template
- Customize Proposal

#### Node: Generate Proposal Copy1
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat generation with structured output.
- **Configuration (interpreted):**
  - Model: `gpt-5-mini`
  - `jsonOutput: true` (expects JSON output)
  - Prompting pattern:
    - System message: sets persona (“senior automation consultant”), tone constraints, anti-hype rules.
    - Instruction message: **must return a single valid JSON object** matching a given schema.
    - Example input JSON and example assistant output JSON included (few-shot).
    - Final user message dynamically builds input JSON from CRM fields:
      - `companyName` from `Company Legal Name`
      - `problem` from `Problem`
      - `solution` from `Solution`
      - `scope` from `Scope`
      - `howSoon` from `How soon?`
      - `depositCost` from `Cost`
      - `currentDate` from `$now.toLocaleString({ dateStyle: 'medium' })`
- **Key expressions / variables:**
  - `{{ $json['Company Legal Name'] }}`
  - `{{ $json.Problem }}`
  - `{{ $json.Solution }}`
  - `{{ $json.Scope }}`
  - `{{ $json['How soon?'] }}`
  - `{{ $json.Cost }}`
  - `{{ $now.toLocaleString({ dateStyle: 'medium' }) }}`
- **Connections:**
  - Input ← **Find Client Details From CRM**
  - Output → **Create Proposal Template**
- **Version-specific notes:** typeVersion `1.6` (LangChain/OpenAI node is versioned; model list and JSON mode behavior can change).
- **Edge cases / failures:**
  - Model may return invalid JSON despite instructions (would break downstream references like `.message.content.proposalTitle`).
  - Missing CRM fields produce malformed input JSON or weak content.
  - OpenAI credential issues, rate limits, timeouts.

#### Node: Create Proposal Template
- **Type / role:** `Google Drive` — copies a Google Slides template file to create a new proposal deck.
- **Configuration (interpreted):**
  - Operation: **Copy**
  - Source file: a specific Slides template file (by file ID).
  - New file name: `={{ $json.message.content.proposalTitle }}`
    - Uses AI output from the previous node.
  - Option: `copyRequiresWriterPermission: false`
- **Key expressions / variables:**
  - `={{ $json.message.content.proposalTitle }}`
- **Connections:**
  - Input ← **Generate Proposal Copy1**
  - Output → **Customize Proposal**
- **Version-specific notes:** typeVersion `3`
- **Edge cases / failures:**
  - Google Drive permission denied on template file.
  - AI output missing `proposalTitle` (name expression fails).
  - Copy succeeds but returns unexpected structure; downstream expects `id`.

#### Node: Customize Proposal
- **Type / role:** `Google Slides` — replace placeholder text tokens in the copied presentation with generated content.
- **Configuration (interpreted):**
  - Operation: **Replace Text**
  - Presentation ID: `={{ $json.id }}` (the copied file ID from Drive copy output)
  - Replacements: a set of placeholder tokens like `{{proposalTitle}}`, `{{solutionHeadingOne}}`, etc.
  - Most replacement values come from:
    - `$('Generate Proposal Copy1').item.json.message.content.<field>`
  - One value is hard-coded:
    - `{{cost}}` → `"$1,850"` (note: not mapped from CRM `Cost`)
- **Key expressions / variables:**
  - `presentationId: ={{ $json.id }}`
  - Many `replaceText` values:  
    `={{ $('Generate Proposal Copy1').item.json.message.content.solutionHeadingOne }}` (and similar)
  - Hard-coded cost: `"$1,850"`
- **Connections:**
  - Input ← **Create Proposal Template**
  - Output → **Create Stripe Payment Link**
- **Version-specific notes:** typeVersion `2`
- **Edge cases / failures:**
  - If placeholders don’t exist in the template, replacements will do nothing (proposal remains uncustomized).
  - If OpenAI output is not present/invalid, expressions referencing `.message.content.*` fail.
  - Slides API permissions/quota issues.
  - Concurrent edits can cause revision/control errors (Slides API write control).

---

### Block 3 — Payment & Follow-up

**Overview (role):**  
Creates a Stripe Checkout Session and emails the lead a follow-up containing the Google Slides proposal link and the Stripe checkout URL.

**Nodes involved:**
- Create Stripe Payment Link
- Email Follow-up

#### Node: Create Stripe Payment Link
- **Type / role:** `HTTP Request` — direct Stripe API call to create a Checkout Session.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.stripe.com/v1/checkout/sessions`
  - Auth: **Stripe predefined credential type** (`stripeApi`)
  - Body: `application/x-www-form-urlencoded`
  - Parameters include:
    - `mode=payment`
    - `success_url=https://www.google.com/success`
    - `cancel_url=https://www.google.com/cancel`
    - `line_items[0][price_data][unit_amount]=10000`
    - `line_items[0][price_data][currency]=usd`
    - `line_items[0][price_data][product_data][name]=Service Package`
    - `line_items[0][quantity]=1`
    - `metadata[leadId]=12345`
- **Key observations:**
  - **Amount is hard-coded** to 10000 (i.e., $100.00 if currency has 2 decimals), not tied to CRM `Cost`.
  - `metadata[leadId]` is hard-coded, not derived from Sheets row_number or a CRM ID.
  - Success/cancel URLs are placeholders (Google).
- **Connections:**
  - Input ← **Customize Proposal**
  - Output → **Email Follow-up**
- **Version-specific notes:** typeVersion `4.3`
- **Edge cases / failures:**
  - Stripe auth errors (invalid secret key, wrong environment).
  - Stripe parameter issues (currency/amount formatting, disallowed settings).
  - Using test keys in production or vice versa.
  - Hard-coded values cause incorrect billing if not customized.

#### Node: Email Follow-up
- **Type / role:** `Gmail` — sends the follow-up message to the lead.
- **Configuration (interpreted):**
  - To: `={{ $('Find Client Details From CRM').item.json.Email }}`
  - Subject: `"Re: Proposal for "` (note: no company/person appended)
  - Body (text email) includes:
    - Personalization: `Person Name` from CRM
    - Proposal link built from the copied Slides file ID:
      - `https://docs.google.com/presentation/d/{{ $('Create Proposal Template').item.json.id }}/edit`
    - Stripe payment link from the immediately previous node output:
      - `{{ $json.url }}`
  - Attribution disabled: `appendAttribution: false`
- **Key expressions / variables:**
  - `sendTo: ={{ $('Find Client Details From CRM').item.json.Email }}`
  - Proposal URL uses: `$('Create Proposal Template').item.json.id`
  - Payment URL uses: `{{ $json.url }}` (Stripe session URL)
- **Connections:**
  - Input ← **Create Stripe Payment Link**
  - Output: none (end)
- **Version-specific notes:** typeVersion `2.1`
- **Edge cases / failures:**
  - Gmail OAuth token expired / insufficient scopes.
  - Missing `Email` or `Person Name` in CRM row.
  - Slides file not shared with recipient (they may get access denied).
  - Subject line incomplete (may reduce clarity/open rate).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow header/overview text |  |  | # Meeting → Proposal → Payment → Follow-up Automation; Automatically turn booked calls into proposals, Stripe payment links, and follow-up emails.; Tools: n8n · OpenAI · Google Sheets · Google Slides · Stripe · Email · Calendly / Forms; Best for: Agencies, consultants, freelancers, and service businesses selling repeatable offers. |
| Calendly Trigger Booked Meeting | Calendly Trigger | Entry point: meeting booked webhook |  | Find Client Details From CRM | ## Trigger & lead context; Starts when a meeting is booked (Calendly); Looks up lead details in a CRM (Google Sheets); Normalizes inputs for downstream automation; This section prepares clean, structured data for proposal generation. |
| Find Client Details From CRM | Google Sheets | CRM lookup/enrichment by invitee email | Calendly Trigger Booked Meeting | Generate Proposal Copy1 | ## Trigger & lead context; Starts when a meeting is booked (Calendly); Looks up lead details in a CRM (Google Sheets); Normalizes inputs for downstream automation; This section prepares clean, structured data for proposal generation. |
| Generate Proposal Copy1 | OpenAI (LangChain) | Generate structured proposal JSON from CRM fields | Find Client Details From CRM | Create Proposal Template | ## Proposal generation & document creation; Uses AI to generate structured proposal content; Copies a Google Slides template; Injects generated content into the deck; Output: a client-ready proposal link with no manual editing. |
| Create Proposal Template | Google Drive | Copy Slides template and name it from AI title | Generate Proposal Copy1 | Customize Proposal | ## Proposal generation & document creation; Uses AI to generate structured proposal content; Copies a Google Slides template; Injects generated content into the deck; Output: a client-ready proposal link with no manual editing. |
| Customize Proposal | Google Slides | Replace placeholders in copied deck with AI content | Create Proposal Template | Create Stripe Payment Link | ## Proposal generation & document creation; Uses AI to generate structured proposal content; Copies a Google Slides template; Injects generated content into the deck; Output: a client-ready proposal link with no manual editing. |
| Create Stripe Payment Link | HTTP Request | Create Stripe Checkout Session | Customize Proposal | Email Follow-up | ## Payment & follow-up; Creates a Stripe Checkout session; Sends a personalized follow-up email; Includes proposal link and payment link; This closes the loop from call → proposal → payment. |
| Email Follow-up | Gmail | Send proposal + payment link email | Create Stripe Payment Link |  | ## Payment & follow-up; Creates a Stripe Checkout session; Sends a personalized follow-up email; Includes proposal link and payment link; This closes the loop from call → proposal → payment. |
| Sticky Note2 | Sticky Note | Comment block: trigger & context |  |  | ## Trigger & lead context; Starts when a meeting is booked (Calendly); Looks up lead details in a CRM (Google Sheets); Normalizes inputs for downstream automation; This section prepares clean, structured data for proposal generation. |
| Sticky Note4 | Sticky Note | Comment block: proposal generation & deck |  |  | ## Proposal generation & document creation; Uses AI to generate structured proposal content; Copies a Google Slides template; Injects generated content into the deck; Output: a client-ready proposal link with no manual editing. |
| Sticky Note5 | Sticky Note | Comment block: payment & follow-up |  |  | ## Payment & follow-up; Creates a Stripe Checkout session; Sends a personalized follow-up email; Includes proposal link and payment link; This closes the loop from call → proposal → payment. |
| Sticky Note8 | Sticky Note | Business impact callout |  |  | ## ⚡ BUSINESS IMPACT; Removes manual proposal writing; Eliminates follow-up delays; Standardizes your close process; Speeds up payment collection; Booked call → proposal → payment → done. |
| Sticky Note9 | Sticky Note | Customization suggestions |  |  | ## 🧩 CUSTOMIZATION NOTES; Swap Google Sheets for Airtable, HubSpot, or Notion; Adjust proposal tone entirely via prompt; Extend Stripe metadata for analytics; Add reminders or follow-ups easily |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Meeting → Proposal → Payment → Follow-up Automation* (or your preferred name).
   - (Optional) Add a tag like “Business Ops”.

2. **Add trigger: “Calendly Trigger”**
   - Node: **Calendly Trigger**
   - Event: **invitee.created**
   - Authentication: **OAuth2**
   - Create/attach **Calendly OAuth2 credentials** in n8n.
   - Save so n8n can register the webhook (activation typically required for live events).

3. **Add CRM lookup: “Google Sheets”**
   - Node: **Google Sheets**
   - Operation: *Lookup/Search* (filter by column)
   - Document: select your CRM spreadsheet
   - Sheet: select the tab containing leads (e.g., “Sheet1”)
   - Filter:
     - Lookup Column: `Email`
     - Lookup Value expression: `={{ $json.payload.email }}`
   - Attach **Google Sheets OAuth2 credentials** with access to the spreadsheet.
   - Connect: **Calendly Trigger → Google Sheets**

4. **Add AI generation: “OpenAI (LangChain) - Chat”**
   - Node: **OpenAI** (`@n8n/n8n-nodes-langchain.openAi`)
   - Model: **gpt-5-mini** (or available equivalent)
   - Enable/require **JSON output**.
   - Messages:
     1) System message defining tone/persona and constraints  
     2) Instruction message: must output one JSON object matching your schema  
     3) (Optional but recommended) include a sample input + sample output (few-shot)  
     4) Final user message: dynamic JSON built from the Sheets row, like:
        - `companyName`: `{{ $json['Company Legal Name'] }}`
        - `problem`: `{{ $json.Problem }}`
        - `solution`: `{{ $json.Solution }}`
        - `scope`: `{{ $json.Scope }}`
        - `howSoon`: `{{ $json['How soon?'] }}`
        - `depositCost`: `{{ $json.Cost }}`
        - `currentDate`: `{{ $now.toLocaleString({ dateStyle: 'medium' }) }}`
   - Attach **OpenAI API credentials**.
   - Connect: **Google Sheets → OpenAI**

5. **Copy the proposal Slides template: “Google Drive”**
   - Node: **Google Drive**
   - Operation: **Copy**
   - File to copy: select your Google Slides template file
   - New name expression: `={{ $json.message.content.proposalTitle }}`
   - Attach **Google Drive OAuth2 credentials**.
   - Connect: **OpenAI → Google Drive**

6. **Inject content into the Slides deck: “Google Slides”**
   - Node: **Google Slides**
   - Operation: **Replace Text**
   - Presentation ID expression: `={{ $json.id }}` (the ID returned by the Drive copy)
   - Add placeholder mappings matching your template tokens, for example:
     - Replace `{{proposalTitle}}` with `={{ $('Generate Proposal Copy1').item.json.message.content.proposalTitle }}`
     - Repeat for each field in your AI schema.
   - Decide how to handle cost:
     - Either keep a fixed value (as in the workflow) or map from CRM/AI output.
   - Attach **Google Slides OAuth2 credentials**.
   - Connect: **Google Drive → Google Slides**

7. **Create Stripe Checkout Session: “HTTP Request”**
   - Node: **HTTP Request**
   - Method: `POST`
   - URL: `https://api.stripe.com/v1/checkout/sessions`
   - Authentication: **Predefined Credential Type → Stripe**
   - Content-Type: `application/x-www-form-urlencoded`
   - Body parameters (minimum viable):
     - `mode`: `payment`
     - `success_url`: your success page URL
     - `cancel_url`: your cancel page URL
     - `line_items[0][price_data][currency]`: `usd` (or your currency)
     - `line_items[0][price_data][unit_amount]`: amount in cents (e.g., `250000` for $2,500.00)
     - `line_items[0][price_data][product_data][name]`: product/service name
     - `line_items[0][quantity]`: `1`
     - `metadata[leadId]`: set dynamically (recommended), e.g. row_number or a CRM ID
   - Attach **Stripe credentials** (test or live).
   - Connect: **Google Slides → HTTP Request**

8. **Send follow-up email: “Gmail”**
   - Node: **Gmail**
   - Operation: **Send**
   - To expression: `={{ $('Find Client Details From CRM').item.json.Email }}`
   - Subject: customize (recommended), e.g. `Re: Proposal for {{ $('Find Client Details From CRM').item.json['Company Legal Name'] }}`
   - Message body includes:
     - Proposal URL: `https://docs.google.com/presentation/d/{{ $('Create Proposal Template').item.json.id }}/edit`
     - Stripe URL from Stripe response: `{{ $json.url }}`
   - Attach **Gmail OAuth2 credentials**.
   - Connect: **HTTP Request → Gmail**

9. **(Optional) Add sticky notes / documentation**
   - Add sticky notes to label the three blocks (Trigger/Context, Proposal/Deck, Payment/Email) and include customization notes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Booked call → proposal → payment → done.” | Business impact / positioning note from the workflow canvas |
| Swap Google Sheets for Airtable, HubSpot, or Notion | Customization note (data source flexibility) |
| Adjust proposal tone entirely via prompt | Customization note (OpenAI prompt controls output tone/structure) |
| Extend Stripe metadata for analytics | Customization note (improve attribution/reporting) |
| Add reminders or follow-ups easily | Customization note (extend flow with delays/secondary emails) |