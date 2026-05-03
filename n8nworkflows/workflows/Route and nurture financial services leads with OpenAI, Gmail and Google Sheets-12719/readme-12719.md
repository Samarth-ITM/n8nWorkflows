Route and nurture financial services leads with OpenAI, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/route-and-nurture-financial-services-leads-with-openai--gmail-and-google-sheets-12719


# Route and nurture financial services leads with OpenAI, Gmail and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow captures inbound leads for financial services (via a landing-page POST webhook), routes each lead by their selected interest, generates a highly personalized HTML email using OpenAI, sends it via Gmail, and logs the lead + key service details into Google Sheets.

**Target use cases:**
- Immediate, personalized first-response for inbound leads (speed-to-lead).
- Service-specific qualification messaging (funding estimates, premium ranges, timelines).
- Centralized lead tracking in a spreadsheet for follow-up.

### 1.1 Input Reception (Lead Capture)
Receives lead payloads from an external form or landing page via a webhook.

### 1.2 Routing (Interest-Based Switch)
Routes the lead to one of four service paths based on `body.interest`.

### 1.3 AI Processing (Email Generation)
Each path uses GPT-4o-mini to generate raw HTML email content, including “calculations” based on lead data.

### 1.4 Email Delivery (Gmail)
Sends the generated HTML email to the lead with a dynamic subject line.

### 1.5 Data Tracking (Google Sheets)
Appends lead/contact data and key service-specific fields to a Google Sheet (same sheet target in this JSON).

---

## 2. Block-by-Block Analysis

### Block A — Input Reception (Webhook)

**Overview:**  
Accepts HTTP POST requests containing lead details and service-specific fields. This is the single entry point for the workflow.

**Nodes involved:**
- `Webhook - Lead Capture`

#### Node: Webhook - Lead Capture
- **Type / role:** `n8n-nodes-base.webhook` — inbound trigger (HTTP endpoint).
- **Configuration (interpreted):**
  - **Method:** POST  
  - **Path:** `your-webhook-path-here` (placeholder; must be replaced)
  - Expects JSON body with `firstName`, `lastName`, `email`, `phone`, `interest`, and `additionalData` (shape depends on interest).
- **Key variables/expressions used downstream:**
  - `$json.body.interest`
  - `$json.body.firstName`, `$json.body.lastName`, `$json.body.email`, `$json.body.phone`
  - `$json.body.additionalData.*`
- **Connections:**
  - Output → `Route by Interest`
- **Edge cases / failures:**
  - Missing or malformed JSON body (downstream expressions will resolve to `undefined`).
  - Wrong field names (e.g., `interest` not in `body`) causes routing to fail and no branch will match.
  - If webhook path/id not configured or not activated, external form submissions will fail.
- **Version notes:** TypeVersion `2.1` — ensure your n8n supports this webhook node version.

---

### Block B — Routing Logic (Switch by interest)

**Overview:**  
Splits the workflow into one of four paths based on exact string matches of the lead’s `interest`.

**Nodes involved:**
- `Route by Interest`

#### Node: Route by Interest
- **Type / role:** `n8n-nodes-base.switch` — conditional router.
- **Configuration (interpreted):**
  - Evaluates `={{ $json.body.interest }}` with **string equals** comparisons.
  - Rules (in order):
    1. `"Business Funding"`
    2. `"Life Insurance"`
    3. `"Credit Repair"`
    4. `"Become an Agent"`
- **Connections:**
  - Output 0 → `AI - Business Funding Email`
  - Output 1 → `AI - Life Insurance Email`
  - Output 2 → `AI - Credit Repair Email`
  - Output 3 → `AI - Recruitment Email`
- **Edge cases / failures:**
  - Interest value must match exactly (case-sensitive); any other value will produce **no output** (lead is effectively dropped).
  - If you also intend to support `"Business Credit Repair"`, note: this switch currently does **not** route that value anywhere (even though the credit-repair prompt supports it).
- **Version notes:** TypeVersion `3.4` — switch condition UI/behavior can differ across versions; keep strict validation in mind.

---

### Block C1 — Business Funding Path (AI → Gmail → Sheets)

**Overview:**  
Generates a funding-focused HTML email with a computed funding range and sends it to the lead, then logs the lead into Google Sheets.

**Nodes involved:**
- `AI - Business Funding Email`
- `Gmail - Business Funding`
- `Sheets - Business Funding`

#### Node: AI - Business Funding Email
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via LangChain wrapper.
- **Configuration (interpreted):**
  - **Model:** `gpt-4o-mini`
  - Prompt builds a detailed email using lead fields from `Webhook - Lead Capture`:
    - business length, monthly revenue, credit score, funding purpose
  - Enforces:
    - output **ONLY raw HTML**
    - include “funding calculation” logic and urgency
    - 350–400 words
  - Contains placeholder CTA link: `YOUR_FUNDING_LINK_HERE` and placeholders like “Your Name/Company”.
- **Key expressions:**
  - Uses many references like:  
    `{{ $('Webhook - Lead Capture').item.json.body.additionalData.monthlyRevenue }}`
- **Connections:**
  - Input ← `Route by Interest` (Business Funding branch)
  - Output → `Gmail - Business Funding`
- **Edge cases / failures:**
  - Missing `additionalData` fields (creditScore/monthlyRevenue/etc.) will reduce personalization and may break the “be specific” requirement.
  - Model may occasionally not follow “raw HTML only” instruction; Gmail will then send undesirable content.
  - OpenAI credential missing/invalid, rate limits, or timeouts.
- **Version notes:** TypeVersion `2` — requires the LangChain OpenAI node package available in your n8n environment.

#### Node: Gmail - Business Funding
- **Type / role:** `n8n-nodes-base.gmail` — sends email via Gmail.
- **Configuration (interpreted):**
  - **To:** lead email from webhook: `={{ $('Webhook - Lead Capture').item.json.body.email }}`
  - **Subject:** `💰 Your Business Funding Pre-Approval - {firstName}`
  - **Message body:** `={{ $json.output[0].content[0].text }}` (expects the OpenAI node output structure)
  - **Option:** `appendAttribution: false` (removes n8n attribution)
- **Connections:**
  - Input ← `AI - Business Funding Email`
  - Output → `Sheets - Business Funding`
- **Edge cases / failures:**
  - Gmail OAuth2 not connected or missing required scopes.
  - `$json.output[0].content[0].text` may differ depending on node/version; if structure changes, message could be blank.
  - Invalid recipient email.
- **Version notes:** TypeVersion `2.1`.

#### Node: Sheets - Business Funding
- **Type / role:** `n8n-nodes-base.googleSheets` — append tracking row.
- **Configuration (interpreted):**
  - **Operation:** Append row
  - **Document:** `YOUR_GOOGLE_SHEET_ID` (placeholder)
  - **Sheet:** `gid=0` (Sheet1)
  - **Columns mapped:**
    - Name, Email, Phone, Details (credit/revenue/length), Interest
- **Connections:**
  - Input ← `Gmail - Business Funding`
  - Output: none
- **Edge cases / failures:**
  - Wrong Sheet ID, missing permissions, Google OAuth issues.
  - Column headers must exist or mapping may not behave as expected (depending on n8n sheets mode).
- **Version notes:** TypeVersion `4.7`.

---

### Block C2 — Life Insurance Path (AI → Gmail → Sheets)

**Overview:**  
Creates a life insurance email with premium range logic based on age and health, sends it, and logs the lead.

**Nodes involved:**
- `AI - Life Insurance Email`
- `Gmail - Life Insurance`
- `Sheets - Life Insurance`

#### Node: AI - Life Insurance Email
- **Type / role:** LangChain OpenAI node — generates HTML email.
- **Configuration (interpreted):**
  - **Model:** `gpt-4o-mini`
  - Uses lead data:
    - ageRange, dependents, healthStatus, coverageNeeded, state
  - Calculates premium range based on age band and health adjustment.
  - Requires raw HTML only, warm tone, not overly salesy.
  - Placeholder CTA link: `YOUR_INSURANCE_LINK_HERE`.
- **Connections:** Input ← Switch (Life Insurance); Output → Gmail.
- **Edge cases:** missing ageRange/healthStatus leads to weak/incorrect premium estimates; OpenAI policy/format drift.
- **Version notes:** TypeVersion `2`.

#### Node: Gmail - Life Insurance
- **Type / role:** Gmail send.
- **Configuration:**
  - To: webhook email
  - Subject: `🛡️ Your Life Insurance Quote - {firstName}`
  - Message: `={{ $json.output[0].content[0].text }}`
  - appendAttribution disabled
- **Connections:** Output → `Sheets - Life Insurance`
- **Edge cases:** auth/scopes, output structure mismatch, deliverability.
- **Version notes:** TypeVersion `2.1`.

#### Node: Sheets - Life Insurance
- **Type / role:** Google Sheets append.
- **Configuration:**
  - Append to `YOUR_GOOGLE_SHEET_ID`, `gid=0`
  - Columns: Age, Name, Email, Phone, State, Interest, Health Status, Coverage Needed
- **Connections:** none after append.
- **Edge cases:** headers/permissions/sheet mismatch.
- **Version notes:** TypeVersion `4.7`.

---

### Block C3 — Credit Repair Path (AI → Gmail → Sheets)

**Overview:**  
Generates a credit repair plan email (supports personal vs business credit repair language inside the prompt), sends it, and logs the lead.

**Nodes involved:**
- `AI - Credit Repair Email`
- `Gmail - Credit Repair`
- `Sheets - Credit Repair`

#### Node: AI - Credit Repair Email
- **Type / role:** LangChain OpenAI node.
- **Configuration (interpreted):**
  - **Model:** `gpt-4o-mini`
  - Uses:
    - `currentScore`, `creditGoals`, `negativeItems`, plus `interest`
  - Prompt includes two modes:
    - Personal credit repair when interest is `"Credit Repair"`
    - Business credit repair when interest is `"Business Credit Repair"` (handled via inline conditional text in HTML template and CTA link selection)
- **Important integration note:**  
  The **Switch routes only `"Credit Repair"`**, not `"Business Credit Repair"`. If your form can send `"Business Credit Repair"`, those leads currently will not reach this node.
- **Connections:** Input ← Switch (Credit Repair); Output → Gmail.
- **Edge cases:** score not numeric/consistent with timeline rules; goal strings not matching expected values; format drift.
- **Version notes:** TypeVersion `2`.

#### Node: Gmail - Credit Repair
- **Type / role:** Gmail send.
- **Configuration:**
  - To: webhook email
  - Subject uses conditional:
    - If interest is `'Business Credit Repair'` → `📈 Build Your Business Credit Fast - {firstName}`
    - Else → `✨ Your Credit Repair Roadmap - {firstName}`
  - Message: `={{ $json.output[0].content[0].text }}`
- **Connections:** Output → `Sheets - Credit Repair`
- **Edge cases:** subject conditional won’t matter if those leads never route here; output structure mismatch.
- **Version notes:** TypeVersion `2.1`.

#### Node: Sheets - Credit Repair
- **Type / role:** Google Sheets append.
- **Configuration:**
  - Append to `YOUR_GOOGLE_SHEET_ID`, `gid=0`
  - Columns: Name, Email, Phone, Details (score + negativeItems), Interest
- **Connections:** none after append.
- **Edge cases:** sheet permissions/headers.
- **Version notes:** TypeVersion `4.7`.

---

### Block C4 — Recruitment Path (AI → Gmail → Sheets)

**Overview:**  
Generates a recruiting email for potential agents, sends it, and logs the lead.

**Nodes involved:**
- `AI - Recruitment Email`
- `Gmail - Recruitment`
- `Sheets - Recruitment`

#### Node: AI - Recruitment Email
- **Type / role:** LangChain OpenAI node.
- **Configuration (interpreted):**
  - **Model:** `gpt-4o-mini`
  - Uses:
    - hasExperience, isLicensed, interests, startTime
  - Output instruction says “raw HTML only”; however the template includes emoji and a casual tone.
  - Placeholder CTA link: `YOUR_RECRUITMENT_LINK_HERE`.
- **Connections:** Input ← Switch (Become an Agent); Output → Gmail.
- **Edge cases:** data fields like `interests` may be array/string; prompt expects specific values; HTML compliance.
- **Version notes:** TypeVersion `2`.

#### Node: Gmail - Recruitment
- **Type / role:** Gmail send.
- **Configuration:**
  - To: webhook email
  - Subject: `🚀 Opportunity Awaits - {firstName}`
  - Message: `={{ $json.output[0].content[0].text }}`
- **Connections:** Output → `Sheets - Recruitment`
- **Edge cases:** OAuth/scopes, output structure mismatch.
- **Version notes:** TypeVersion `2.1`.

#### Node: Sheets - Recruitment
- **Type / role:** Google Sheets append.
- **Configuration:**
  - Append to `YOUR_GOOGLE_SHEET_ID`, `gid=0`
  - Columns: Name, Email, Phone, Interest
- **Connections:** none after append.
- **Edge cases:** sheet permissions/headers.
- **Version notes:** TypeVersion `4.7`.

---

### Block D — Documentation / Notes (Sticky Notes)

**Overview:**  
These nodes are non-executing annotations that describe setup, expected payloads, and extension ideas.

**Nodes involved:**
- `Main Overview`
- `Webhook Data`
- `Routing Logic`
- `AI Generation`
- `Email Delivery`
- `Data Tracking`
- `Extensions`

#### Sticky notes (technical relevance)
- They do not affect execution, but contain important setup guidance and assumptions (e.g., payload schema).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Lead Capture | Webhook | Entry point: receive lead JSON via POST | — | Route by Interest | ## How it works … Built by David Olusola \| DaexAI / ## Webhook expects … additionalData fields… |
| Route by Interest | Switch | Route lead to correct service branch by `interest` | Webhook - Lead Capture | AI - Business Funding Email; AI - Life Insurance Email; AI - Credit Repair Email; AI - Recruitment Email | ## Routes by interest … 4 paths… |
| AI - Business Funding Email | OpenAI (LangChain) | Generate personalized funding HTML email | Route by Interest | Gmail - Business Funding | ## AI email generation … outputs raw HTML… customize prompts… |
| Gmail - Business Funding | Gmail | Send funding email | AI - Business Funding Email | Sheets - Business Funding | ## Gmail automation … Remove attribution … Test deliverability… |
| Sheets - Business Funding | Google Sheets | Append funding lead row | Gmail - Business Funding | — | ## Lead logging … separate sheets or filtered views… |
| AI - Life Insurance Email | OpenAI (LangChain) | Generate personalized insurance HTML email | Route by Interest | Gmail - Life Insurance | ## AI email generation … outputs raw HTML… customize prompts… |
| Gmail - Life Insurance | Gmail | Send insurance email | AI - Life Insurance Email | Sheets - Life Insurance | ## Gmail automation … Remove attribution … Test deliverability… |
| Sheets - Life Insurance | Google Sheets | Append insurance lead row | Gmail - Life Insurance | — | ## Lead logging … separate sheets or filtered views… |
| AI - Credit Repair Email | OpenAI (LangChain) | Generate personalized credit repair HTML email | Route by Interest | Gmail - Credit Repair | ## AI email generation … outputs raw HTML… customize prompts… |
| Gmail - Credit Repair | Gmail | Send credit repair email | AI - Credit Repair Email | Sheets - Credit Repair | ## Gmail automation … Remove attribution … Test deliverability… |
| Sheets - Credit Repair | Google Sheets | Append credit repair lead row | Gmail - Credit Repair | — | ## Lead logging … separate sheets or filtered views… |
| AI - Recruitment Email | OpenAI (LangChain) | Generate recruiting HTML email | Route by Interest | Gmail - Recruitment | ## AI email generation … outputs raw HTML… customize prompts… |
| Gmail - Recruitment | Gmail | Send recruitment email | AI - Recruitment Email | Sheets - Recruitment | ## Gmail automation … Remove attribution … Test deliverability… |
| Sheets - Recruitment | Google Sheets | Append recruitment lead row | Gmail - Recruitment | — | ## Lead logging … separate sheets or filtered views… |
| Main Overview | Sticky Note | Documentation / setup guidance | — | — | (content as shown in node) |
| Webhook Data | Sticky Note | Documents expected webhook payload schema | — | — | (content as shown in node) |
| Routing Logic | Sticky Note | Documents routing branches | — | — | (content as shown in node) |
| AI Generation | Sticky Note | Documents AI generation approach | — | — | (content as shown in node) |
| Email Delivery | Sticky Note | Documents Gmail sending considerations | — | — | (content as shown in node) |
| Data Tracking | Sticky Note | Documents Sheets logging approach | — | — | (content as shown in node) |
| Extensions | Sticky Note | Ideas for extending the automation | — | — | (content as shown in node) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Trigger Node: Webhook**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: choose your own (replace `your-webhook-path-here`)
   - Save and copy the **Test URL** and **Production URL**.
   - Update your landing page/form to POST JSON to this webhook URL with:
     - `firstName`, `lastName`, `email`, `phone`, `interest`, `additionalData` (service-specific).

3. **Add Router Node: Switch**
   - Node type: **Switch**
   - Value to evaluate: `{{ $json.body.interest }}`
   - Create 4 rules (String → Equals), case-sensitive:
     - `Business Funding`
     - `Life Insurance`
     - `Credit Repair`
     - `Become an Agent`
   - Connect: **Webhook → Switch**

4. **Create Business Funding branch**
   1. Add node: **OpenAI (LangChain)**
      - Node type: `@n8n/n8n-nodes-langchain.openAi`
      - Model: **gpt-4o-mini**
      - Configure the prompt to reference the webhook fields (firstName, revenue, credit, etc.)
      - Ensure it outputs **raw HTML** (no markdown/fences).
      - Credentials: add your **OpenAI API credential** in n8n and select it here.
   2. Add node: **Gmail**
      - Operation: **Send**
      - To: `{{ $('Webhook - Lead Capture').item.json.body.email }}`
      - Subject: `💰 Your Business Funding Pre-Approval - {{ firstName }}`
      - Message body: map from the OpenAI node’s output (test-run once to confirm the exact output path; in this workflow it is `{{$json.output[0].content[0].text}}`)
      - Option: disable attribution footer (appendAttribution = false)
      - Credentials: connect **Gmail OAuth2** (ensure send scope).
   3. Add node: **Google Sheets**
      - Operation: **Append**
      - Document: select your spreadsheet (store its ID)
      - Sheet: choose the target tab (e.g., Sheet1)
      - Map columns (Name, Email, Phone, Details, Interest) using webhook expressions.
   4. Connect: **Switch (Business Funding output) → OpenAI → Gmail → Sheets**

5. **Create Life Insurance branch**
   - Repeat the same pattern:
     - **OpenAI node** with insurance prompt/premium logic (ageRange, healthStatus, etc.)
     - **Gmail send** with subject `🛡️ Your Life Insurance Quote - {firstName}`
     - **Google Sheets append** with insurance columns (Age, State, Coverage Needed, etc.)
   - Connect: **Switch (Life Insurance output) → OpenAI → Gmail → Sheets**

6. **Create Credit Repair branch**
   - Add **OpenAI node** with credit repair prompt (currentScore, goals, negativeItems).
   - Add **Gmail send** with conditional subject if desired.
   - Add **Google Sheets append** with credit repair details.
   - Connect: **Switch (Credit Repair output) → OpenAI → Gmail → Sheets**
   - If you need **Business Credit Repair** as a separate interest value:
     - Either add a 5th Switch rule for `"Business Credit Repair"` routing to the same AI node, or normalize interest values upstream.

7. **Create Recruitment branch**
   - Add **OpenAI node** with recruiting prompt (experience, licensed, interests, startTime).
   - Add **Gmail send** with subject `🚀 Opportunity Awaits - {firstName}`
   - Add **Google Sheets append** with basic contact columns.
   - Connect: **Switch (Become an Agent output) → OpenAI → Gmail → Sheets**

8. **Replace placeholders inside prompts**
   - Replace:
     - `Your Name`, `Your Company`, `Your Title`
     - `YOUR_FUNDING_LINK_HERE`, `YOUR_INSURANCE_LINK_HERE`, `YOUR_CREDIT_REPAIR_LINK_HERE`, `YOUR_BUSINESS_CREDIT_LINK_HERE`, `YOUR_RECRUITMENT_LINK_HERE`

9. **Test each path**
   - Use the webhook’s **Test** mode and POST sample JSON for each interest.
   - Confirm:
     - Switch routes correctly
     - OpenAI returns HTML
     - Gmail sends correctly
     - Sheets rows append successfully

10. **Activate workflow**
   - Switch webhook to production usage and update your landing page to the production webhook URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Built by David Olusola \| DaexAI” | From the “Main Overview” note |
| The workflow expects a JSON body with top-level identity fields plus `additionalData` depending on the selected service. | From “Webhook Data” note |
| Gmail nodes disable attribution footer and advise deliverability testing. | From “Email Delivery” note |
| You can extend with SMS, CRM, follow-ups, Slack alerts, calendar booking; monitor AI costs and A/B test templates. | From “Extensions” note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.