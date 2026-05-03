Automate multi-step onboarding emails from Notion leads via Gmail and Telegram

https://n8nworkflows.xyz/workflows/automate-multi-step-onboarding-emails-from-notion-leads-via-gmail-and-telegram-12633


# Automate multi-step onboarding emails from Notion leads via Gmail and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automate multi-step onboarding emails from Notion leads via Gmail and Telegram  
**Workflow name (in JSON):** Send automated onboarding emails to Notion leads via Gmail  
**Purpose:** Runs daily (or manually) to pull leads/users from a Notion database and send a staged onboarding/drip email sequence via Gmail. It updates Notion checkboxes/status fields to prevent duplicate sends, and sends a Telegram internal alert when a user reaches Day 7 without converting.

### 1.1 Entry & Configuration Block
- Two entry points (Schedule + Manual) feed a single configuration node that centralizes subjects/bodies, manager name, and Telegram chat ID.

### 1.2 Stage 1 – Welcome (Day 0)
- Finds “New” users who haven’t received the welcome email, sends it, then marks the Notion checkbox.

### 1.3 Stage 2 – Pro Tip (Day 1)
- Finds users registered before “now - 1 day” who haven’t received Tip #1, sends it, then marks the checkbox.

### 1.4 Stage 3 – Soft Check-in (Day 3)
- Finds users registered before “now - 3 days” who haven’t received the follow-up, sends it, then marks the checkbox.

### 1.5 Stage 4 – Sales Push (Day 7) + Telegram Alert
- Finds users registered before “now - 7 days” who haven’t received the personal outreach, sends it, marks it, then notifies the team in Telegram.

### 1.6 Stage 5 – Trial Expiry (3 days before)
- Finds trial users whose expiry date is before “now + 3 days” and who haven’t been alerted, sends a payment reminder, then updates subscription status and marks alert checkbox.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Central Configuration
**Overview:** Provides two ways to run the workflow and centralizes all copy/IDs used by downstream Gmail and Telegram nodes.  
**Nodes involved:** `Run Daily`, `Manual Trigger`, `📝 CONFIGURATION`

#### Node: Run Daily
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) – automated entry point.
- **Configuration:** Runs every 24 hours (`hoursInterval: 24`).
- **Connections:** Outputs to `📝 CONFIGURATION`.
- **Version:** 1.2
- **Edge cases / failures:**
  - Timezone considerations: schedule uses the instance/server timezone; “daily” may not align with your business day.
  - If execution overlaps (long runs), may create parallel runs (depends on n8n settings).

#### Node: Manual Trigger
- **Type / role:** Manual Trigger – testing entry point.
- **Configuration:** Default.
- **Connections:** Outputs to `📝 CONFIGURATION`.
- **Version:** 1
- **Edge cases:** None (manual runs can still hit API limits downstream).

#### Node: 📝 CONFIGURATION
- **Type / role:** Set node – stores constants used throughout the workflow.
- **Notes (sticky guidance on node):** “👇 EDIT EMAILS HERE”
- **Configuration choices (interpreted):**
  - Defines string fields:
    - `PRODUCT_NAME`, `MANAGER_NAME`
    - `TELEGRAM_CHAT_ID` (currently `-1+1234567890` which is *not* a typical Telegram chat ID format; see edge cases)
    - Email subjects and bodies for Welcome/Day1/Day3/Day7/Payment
  - Email bodies include HTML (`<br>`).
- **Key expressions used by other nodes:**
  - Downstream nodes read values via: `$('📝 CONFIGURATION').first().json.<KEY>`
- **Connections:**
  - **Input:** from `Run Daily` and `Manual Trigger`
  - **Output:** fan-out to all “Get … Users” Notion nodes (Stages 1–5)
- **Version:** 2
- **Edge cases / failures:**
  - If this node outputs **no items** (unusual for Set), downstream could break; by default it outputs one item.
  - `TELEGRAM_CHAT_ID` must be a valid numeric ID or `@channelusername`. The provided value looks malformed; Telegram send will fail with “chat not found”.

---

### Block 2 — Stage 1: Welcome (Day 0)
**Overview:** Pulls brand-new leads from Notion, sends the welcome email, then flags them as completed to avoid duplicates.  
**Nodes involved:** `1. Get New Users`, `📧 Send Welcome`, `✅ Mark Welcome`

#### Node: 1. Get New Users
- **Type / role:** Notion – query database pages matching “new + not welcomed”.
- **Notes:** “Step 0: Welcome”
- **Configuration choices:**
  - **Resource/Operation:** Database Page → Get All
  - **Filter (all must match):**
    - `Pipeline Stage|status` equals `New`
    - `Onboarding: Welcome|checkbox` equals **false/unchecked** (implemented as “equals” without a value in JSON; in n8n UI this corresponds to unchecked)
  - **Database:** `databaseId` is empty in the JSON export → must be selected.
- **Connections:** Output → `📧 Send Welcome`
- **Version:** 2.2
- **Edge cases / failures:**
  - Missing database selection causes runtime error.
  - Property names must match exactly in Notion (including punctuation/spaces).
  - If `Email` property is missing or not an Email type, downstream Gmail node will fail.

#### Node: 📧 Send Welcome
- **Type / role:** Gmail – sends the welcome email.
- **Configuration choices:**
  - **To:** `={{ $json.properties.Email.email }}`
  - **Subject:** from configuration: `EMAIL_WELCOME_SUBJ`
  - **Message:** from configuration: `EMAIL_WELCOME_BODY` (HTML content)
  - **Options:** attribution disabled (`appendAttribution: false`)
- **Connections:** Output → `✅ Mark Welcome`
- **Version:** 2.1
- **Edge cases / failures:**
  - Gmail OAuth credential issues (expired token / missing scopes).
  - Invalid or empty recipient email.
  - If Notion returns multiple pages, Gmail node will send one email per item (intended).
  - HTML rendering: Gmail node sends body as-is; ensure the node is configured to send HTML if required by your n8n/Gmail node behavior (in some setups you may need explicit “HTML” option; here it relies on Gmail node defaults).

#### Node: ✅ Mark Welcome
- **Type / role:** Notion – updates the page to mark welcome step completed.
- **Notes:** “⚠️ Select DB”
- **Configuration choices:**
  - **Operation:** Database Page → Update
  - **Page ID:** `={{ $json.id }}`
  - **Properties updated:** `Onboarding: Welcome|checkbox` → true
- **Connections:** none further (end of this branch)
- **Version:** 2.2
- **Edge cases / failures:**
  - If the page ID is missing (unexpected), update fails.
  - Notion permission issues (integration not shared with DB).
  - Property type mismatch (checkbox must be checkbox).

---

### Block 3 — Stage 2: Pro Tip (Day 1)
**Overview:** Targets users older than ~24h who haven’t received the first tip, sends the tip email, then marks completion in Notion.  
**Nodes involved:** `2. Get Day 1 Users`, `📧 Send Tip #1`, `✅ Mark Tip #1`

#### Node: 2. Get Day 1 Users
- **Type / role:** Notion – query users eligible for Day 1 message.
- **Notes:** “Step 1: Tips”
- **Configuration choices:**
  - **Filter (all must match):**
    - `Onboarding: Tip #1|checkbox` unchecked
    - `Registration Date|date` **before** `{{ new Date(Date.now() - 1 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10) }}`
      - This builds a `YYYY-MM-DD` string in UTC.
  - **Database:** not selected in JSON (must be set).
- **Connections:** Output → `📧 Send Tip #1`
- **Version:** 2.2
- **Edge cases / failures:**
  - Timezone mismatch: comparing a UTC date string to Notion date fields can cause off-by-one-day effects depending on how `Registration Date` is stored (date vs datetime).
  - Users registered 2+ days ago will also match (because “before day-1”), which is typically fine in drip logic since checkbox prevents repeats.

#### Node: 📧 Send Tip #1
- **Type / role:** Gmail – sends Day 1 pro tip email.
- **Configuration:** Same pattern as welcome:
  - To: `properties.Email.email`
  - Subject/body from `📝 CONFIGURATION` (`EMAIL_DAY1_SUBJ`, `EMAIL_DAY1_BODY`)
  - Attribution off
- **Connections:** Output → `✅ Mark Tip #1`
- **Version:** 2.1
- **Edge cases:** same as other Gmail nodes.

#### Node: ✅ Mark Tip #1
- **Type / role:** Notion – marks tip as sent.
- **Notes:** “⚠️ Select DB”
- **Configuration:** Update page `id`, set `Onboarding: Tip #1` checkbox true.
- **Connections:** none
- **Version:** 2.2
- **Edge cases:** same as other Notion update nodes.

---

### Block 4 — Stage 3: Soft Check-in (Day 3)
**Overview:** Sends a follow-up email after ~3 days to users who haven’t completed that stage, then marks it in Notion.  
**Nodes involved:** `3. Get Day 3 Users`, `📧 Send Follow-up`, `✅ Mark Follow-up`

#### Node: 3. Get Day 3 Users
- **Type / role:** Notion – query Day 3 eligible users.
- **Notes:** “Step 2: Check-in”
- **Configuration choices:**
  - Filters:
    - `Onboarding: Follow-up|checkbox` unchecked
    - `Registration Date|date` before `now - 3 days` (same UTC date-string approach)
  - Database not set in JSON.
- **Connections:** Output → `📧 Send Follow-up`
- **Version:** 2.2
- **Edge cases:** same timezone/“before” semantics as Day 1.

#### Node: 📧 Send Follow-up
- **Type / role:** Gmail – sends Day 3 email.
- **Configuration:** To = Notion Email; subject/body from config (`EMAIL_DAY3_*`); attribution off.
- **Connections:** Output → `✅ Mark Follow-up`
- **Version:** 2.1
- **Edge cases:** Gmail auth, invalid recipients, rate limits.

#### Node: ✅ Mark Follow-up
- **Type / role:** Notion update – set `Onboarding: Follow-up` to true.
- **Notes:** “⚠️ Select DB”
- **Connections:** none
- **Version:** 2.2

---

### Block 5 — Stage 4: Sales Push (Day 7) + Telegram Notification
**Overview:** After ~7 days, sends a more personal email and alerts the internal team in Telegram about an unconverted lead.  
**Nodes involved:** `4. Get Day 7 Users`, `📧 Send Manager Email`, `✅ Mark Personal`, `📢 Notify Team`

#### Node: 4. Get Day 7 Users
- **Type / role:** Notion – query Day 7 eligible users.
- **Notes:** “Step 3: Personal”
- **Configuration choices:**
  - Filters:
    - `Onboarding: Personal|checkbox` unchecked
    - `Registration Date|date` before `now - 7 days` (UTC date string)
  - Database not set in JSON.
- **Connections:** Output → `📧 Send Manager Email`
- **Version:** 2.2
- **Edge cases:** same as other Notion “Get All” nodes.

#### Node: 📧 Send Manager Email
- **Type / role:** Gmail – sends Day 7 personal email.
- **Configuration:** Uses `EMAIL_DAY7_SUBJ` / `EMAIL_DAY7_BODY`.
- **Connections:** Output → `✅ Mark Personal`
- **Version:** 2.1
- **Edge cases:** same as other Gmail nodes.

#### Node: ✅ Mark Personal
- **Type / role:** Notion update – sets `Onboarding: Personal` true.
- **Notes:** “⚠️ Select DB”
- **Connections:** Output → `📢 Notify Team`
- **Version:** 2.2
- **Edge cases:**
  - If the Notion update fails, Telegram won’t fire (since it’s downstream). If you want Telegram even on update failure, you’d need error handling.

#### Node: 📢 Notify Team
- **Type / role:** Telegram – sends internal notification.
- **Configuration choices:**
  - **Chat ID:** `={{ $('📝 CONFIGURATION').first().json.TELEGRAM_CHAT_ID }}`
  - **Parse mode:** HTML
  - **Text (HTML template):** includes company name pulled from Notion:
    - `{{ $json["properties"]["Company Name"]["title"][0]["plain_text"] }}`
  - Attribution off
- **Connections:** none
- **Version:** 1.2
- **Edge cases / failures:**
  - **Company Name empty:** if `title` array is empty, `[0]` access causes an expression error. Safer pattern would be optional chaining or a fallback.
  - Telegram credential invalid / bot not in chat / wrong chat ID format.
  - If chat is a private group/channel, bot must be added and have permission to post.

---

### Block 6 — Stage 5: Trial Expiry (3 days before) + Status Update
**Overview:** Detects trial users expiring soon, sends an upgrade reminder, then marks them as alerted and changes subscription status.  
**Nodes involved:** `5. Get Expiring Users`, `📧 Send Payment Alert`, `✅ Update Status`

#### Node: 5. Get Expiring Users
- **Type / role:** Notion – query users whose trial is about to expire.
- **Notes:** “Step 4: Payment”
- **Configuration choices:**
  - Filters (all must match):
    - `Subscription Status|status` equals `Trial`
    - `System: Payment Alert|checkbox` unchecked
    - `Expiry Date|date` before `now + 3 days` (UTC date string)
- **Connections:** Output → `📧 Send Payment Alert`
- **Version:** 2.2
- **Edge cases / failures:**
  - The “before now+3 days” logic matches anything expiring sooner than 3 days **and also already expired**; if you only want “within the next 3 days”, you’d typically add an “after today” condition too.
  - Database not set in JSON.

#### Node: 📧 Send Payment Alert
- **Type / role:** Gmail – sends trial expiry reminder email.
- **Configuration:** subject/body from `EMAIL_PAYMENT_*`, recipient from Notion Email, attribution off.
- **Connections:** Output → `✅ Update Status`
- **Version:** 2.1
- **Edge cases:** Gmail auth/limits, invalid email.

#### Node: ✅ Update Status
- **Type / role:** Notion update – marks alert sent + changes subscription status.
- **Notes:** “⚠️ Select DB”
- **Configuration choices:**
  - Update page ID = `{{$json.id}}`
  - Set:
    - `Subscription Status|status` → `Expiring`
    - `System: Payment Alert|checkbox` → true
- **Connections:** none
- **Version:** 2.2
- **Edge cases:**
  - Status value must exist in the Notion status property options (“Expiring” must be defined).
  - Permission/property mismatch errors.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Daily | Schedule Trigger | Daily automated start | — | 📝 CONFIGURATION | # 🚀 SaaS Onboarding Drip; This workflow automates a 5-step email sequence for new users stored in Notion. It handles welcome emails, pro-tips, check-ins, and trial expiration alerts automatically. |
| Manual Trigger | Manual Trigger | Manual start for testing | — | 📝 CONFIGURATION | # 🚀 SaaS Onboarding Drip; This workflow automates a 5-step email sequence for new users stored in Notion. It handles welcome emails, pro-tips, check-ins, and trial expiration alerts automatically. |
| 📝 CONFIGURATION | Set | Central constants for email copy + Telegram target | Run Daily; Manual Trigger | 1. Get New Users; 2. Get Day 1 Users; 3. Get Day 3 Users; 4. Get Day 7 Users; 5. Get Expiring Users | ## 📝 CONFIGURATION; Set all your email subjects and body texts here. Also set your **Telegram Chat ID** for internal notifications. |
| 1. Get New Users | Notion | Query Stage 1 eligible leads | 📝 CONFIGURATION | 📧 Send Welcome | ## Stage 1: Welcome (Day 0); Identifies new signups and immediately sends the welcome email. |
| 📧 Send Welcome | Gmail | Send welcome email | 1. Get New Users | ✅ Mark Welcome | ## Stage 1: Welcome (Day 0); Identifies new signups and immediately sends the welcome email. |
| ✅ Mark Welcome | Notion | Mark welcome sent in Notion | 📧 Send Welcome | — | ## Stage 1: Welcome (Day 0); Identifies new signups and immediately sends the welcome email. |
| 2. Get Day 1 Users | Notion | Query Day 1 eligible users | 📝 CONFIGURATION | 📧 Send Tip #1 | ## Stage 2: Pro Tip (Day 1); Sends a value-add tip 24 hours after registration to increase feature adoption. |
| 📧 Send Tip #1 | Gmail | Send Day 1 tip email | 2. Get Day 1 Users | ✅ Mark Tip #1 | ## Stage 2: Pro Tip (Day 1); Sends a value-add tip 24 hours after registration to increase feature adoption. |
| ✅ Mark Tip #1 | Notion | Mark Day 1 tip sent | 📧 Send Tip #1 | — | ## Stage 2: Pro Tip (Day 1); Sends a value-add tip 24 hours after registration to increase feature adoption. |
| 3. Get Day 3 Users | Notion | Query Day 3 eligible users | 📝 CONFIGURATION | 📧 Send Follow-up | ## Stage 3: Soft Check-in (Day 3); Automatically asks users if they need help setting up their account. |
| 📧 Send Follow-up | Gmail | Send Day 3 follow-up | 3. Get Day 3 Users | ✅ Mark Follow-up | ## Stage 3: Soft Check-in (Day 3); Automatically asks users if they need help setting up their account. |
| ✅ Mark Follow-up | Notion | Mark follow-up sent | 📧 Send Follow-up | — | ## Stage 3: Soft Check-in (Day 3); Automatically asks users if they need help setting up their account. |
| 4. Get Day 7 Users | Notion | Query Day 7 eligible users | 📝 CONFIGURATION | 📧 Send Manager Email | ## Stage 4: Sales Push (Day 7); Sends a "personal" email from a manager and alerts the team via Telegram about unconverted leads. |
| 📧 Send Manager Email | Gmail | Send Day 7 personal email | 4. Get Day 7 Users | ✅ Mark Personal | ## Stage 4: Sales Push (Day 7); Sends a "personal" email from a manager and alerts the team via Telegram about unconverted leads. |
| ✅ Mark Personal | Notion | Mark Day 7 email sent | 📧 Send Manager Email | 📢 Notify Team | ## Stage 4: Sales Push (Day 7); Sends a "personal" email from a manager and alerts the team via Telegram about unconverted leads. |
| 📢 Notify Team | Telegram | Internal “hot lead” alert | ✅ Mark Personal | — | ## Stage 4: Sales Push (Day 7); Sends a "personal" email from a manager and alerts the team via Telegram about unconverted leads. |
| 5. Get Expiring Users | Notion | Query expiring trials | 📝 CONFIGURATION | 📧 Send Payment Alert | ## Stage 5: Trial Expiry; Detects users whose trial ends in 3 days and sends an upgrade reminder. |
| 📧 Send Payment Alert | Gmail | Send trial expiry reminder | 5. Get Expiring Users | ✅ Update Status | ## Stage 5: Trial Expiry; Detects users whose trial ends in 3 days and sends an upgrade reminder. |
| ✅ Update Status | Notion | Mark payment alert + set status Expiring | 📧 Send Payment Alert | — | ## Stage 5: Trial Expiry; Detects users whose trial ends in 3 days and sends an upgrade reminder. |
| Sticky Note | Sticky Note | Comment block (configuration) | — | — | ## 📝 CONFIGURATION; Set all your email subjects and body texts here. Also set your **Telegram Chat ID** for internal notifications. |
| Sticky Note 1 | Sticky Note | Comment block (Stage 1) | — | — | ## Stage 1: Welcome (Day 0); Identifies new signups and immediately sends the welcome email. |
| Sticky Note1 | Sticky Note | Comment block (overall) | — | — | # 🚀 SaaS Onboarding Drip; Includes “How it works” + setup steps (see General Notes). |
| Sticky Note2 | Sticky Note | Comment block (Stage 2) | — | — | ## Stage 2: Pro Tip (Day 1); Sends a value-add tip 24 hours after registration to increase feature adoption. |
| Sticky Note3 | Sticky Note | Comment block (Stage 3) | — | — | ## Stage 3: Soft Check-in (Day 3); Automatically asks users if they need help setting up their account. |
| Sticky Note5 | Sticky Note | Comment block (Stage 4) | — | — | ## Stage 4: Sales Push (Day 7); Sends a "personal" email from a manager and alerts the team via Telegram about unconverted leads. |
| Sticky Note6 | Sticky Note | Comment block (Stage 5) | — | — | ## Stage 5: Trial Expiry; Detects users whose trial ends in 3 days and sends an upgrade reminder. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Trigger nodes (two entry points):**
   1. Add **Schedule Trigger** named `Run Daily`
      - Set interval to **every 24 hours**.
   2. Add **Manual Trigger** named `Manual Trigger` for testing.
3. **Add a Set node** named `📝 CONFIGURATION`
   - Add these String fields (examples from workflow; customize as needed):
     - `PRODUCT_NAME`, `MANAGER_NAME`, `TELEGRAM_CHAT_ID`
     - `EMAIL_WELCOME_SUBJ`, `EMAIL_WELCOME_BODY`
     - `EMAIL_DAY1_SUBJ`, `EMAIL_DAY1_BODY`
     - `EMAIL_DAY3_SUBJ`, `EMAIL_DAY3_BODY`
     - `EMAIL_DAY7_SUBJ`, `EMAIL_DAY7_BODY`
     - `EMAIL_PAYMENT_SUBJ`, `EMAIL_PAYMENT_BODY`
   - Ensure it outputs one item (default behavior).
4. **Connect triggers to config:**
   - Connect `Run Daily` → `📝 CONFIGURATION`
   - Connect `Manual Trigger` → `📝 CONFIGURATION`
5. **Prepare Notion credentials & database:**
   - Create/choose a Notion integration credential in n8n.
   - In Notion, **share** the target database with that integration.
   - Ensure database properties exist and match names used in filters/updates:
     - `Pipeline Stage` (Status)
     - `Email` (Email)
     - `Registration Date` (Date)
     - `Company Name` (Title)
     - `Onboarding: Welcome` (Checkbox)
     - `Onboarding: Tip #1` (Checkbox)
     - `Onboarding: Follow-up` (Checkbox)
     - `Onboarding: Personal` (Checkbox)
     - `Subscription Status` (Status with values incl. `Trial`, `Expiring`)
     - `System: Payment Alert` (Checkbox)
     - `Expiry Date` (Date)
6. **Stage 1 nodes (Welcome):**
   1. Add **Notion** node `1. Get New Users`
      - Resource: **Database Page**, Operation: **Get All**
      - Select your **Database**
      - Add filters (match all):
        - `Pipeline Stage` **equals** `New`
        - `Onboarding: Welcome` **equals** unchecked/false
   2. Add **Gmail** node `📧 Send Welcome`
      - Credential: connect Gmail OAuth2
      - To: `={{ $json.properties.Email.email }}`
      - Subject: `={{ $('📝 CONFIGURATION').first().json.EMAIL_WELCOME_SUBJ }}`
      - Message: `={{ $('📝 CONFIGURATION').first().json.EMAIL_WELCOME_BODY }}`
      - Turn off “append attribution” if available
   3. Add **Notion** node `✅ Mark Welcome`
      - Operation: **Update** database page
      - Page ID: `={{ $json.id }}`
      - Set property `Onboarding: Welcome` to **true**
   4. Connect: `📝 CONFIGURATION` → `1. Get New Users` → `📧 Send Welcome` → `✅ Mark Welcome`
7. **Stage 2 nodes (Day 1 Tip):**
   1. Notion node `2. Get Day 1 Users` (Get All + filters)
      - `Onboarding: Tip #1` unchecked
      - `Registration Date` **before**:  
        `={{ new Date(Date.now() - 1 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10) }}`
   2. Gmail node `📧 Send Tip #1`
      - To: `={{ $json.properties.Email.email }}`
      - Subject/body from `EMAIL_DAY1_SUBJ` / `EMAIL_DAY1_BODY`
   3. Notion node `✅ Mark Tip #1` update checkbox true
   4. Connect: `📝 CONFIGURATION` → `2. Get Day 1 Users` → `📧 Send Tip #1` → `✅ Mark Tip #1`
8. **Stage 3 nodes (Day 3 Follow-up):**
   1. Notion `3. Get Day 3 Users` filters:
      - `Onboarding: Follow-up` unchecked
      - `Registration Date` before:  
        `={{ new Date(Date.now() - 3 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10) }}`
   2. Gmail `📧 Send Follow-up` subject/body from `EMAIL_DAY3_*`
   3. Notion `✅ Mark Follow-up` set checkbox true
   4. Connect: `📝 CONFIGURATION` → `3. Get Day 3 Users` → `📧 Send Follow-up` → `✅ Mark Follow-up`
9. **Stage 4 nodes (Day 7 Personal + Telegram):**
   1. Notion `4. Get Day 7 Users` filters:
      - `Onboarding: Personal` unchecked
      - `Registration Date` before:  
        `={{ new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10) }}`
   2. Gmail `📧 Send Manager Email` subject/body from `EMAIL_DAY7_*`
   3. Notion `✅ Mark Personal` set checkbox true
   4. Telegram `📢 Notify Team`
      - Credential: Telegram bot token
      - Chat ID: `={{ $('📝 CONFIGURATION').first().json.TELEGRAM_CHAT_ID }}`
      - Parse mode: **HTML**
      - Text template (adapt as needed) referencing Notion company name and manager name
   5. Connect: `📝 CONFIGURATION` → `4. Get Day 7 Users` → `📧 Send Manager Email` → `✅ Mark Personal` → `📢 Notify Team`
10. **Stage 5 nodes (Trial Expiry):**
    1. Notion `5. Get Expiring Users` filters:
       - `Subscription Status` equals `Trial`
       - `System: Payment Alert` unchecked
       - `Expiry Date` before:  
         `={{ new Date(Date.now() + 3 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10) }}`
    2. Gmail `📧 Send Payment Alert` subject/body from `EMAIL_PAYMENT_*`
    3. Notion `✅ Update Status`
       - Set `Subscription Status` to `Expiring`
       - Set `System: Payment Alert` checkbox true
    4. Connect: `📝 CONFIGURATION` → `5. Get Expiring Users` → `📧 Send Payment Alert` → `✅ Update Status`
11. **Credentials checklist**
    - **Notion:** Integration token; database shared with integration.
    - **Gmail:** OAuth2 with permission to send email (and from the desired mailbox).
    - **Telegram:** Bot token; bot added to group/channel; valid chat ID or channel username.
12. **Validate with Manual Trigger**
    - Run `Manual Trigger`, inspect each Notion “Get … Users” output, then confirm Gmail/Notion updates.
13. **Activate workflow**
    - Turn workflow to **Active** once verified.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # 🚀 SaaS Onboarding Drip — Automates a 5-step email sequence from Notion, updates CRM fields, and alerts the team on Day 7. Includes setup steps: configure `📝 CONFIGURATION`, connect Notion/Gmail/Telegram credentials, and ensure Notion property names match. | From the workflow’s overall sticky note content |
| ## 📝 CONFIGURATION — Set all email subjects/bodies and Telegram Chat ID in one place. | From configuration sticky note content |
| Stage notes: Welcome (Day 0), Pro Tip (Day 1), Soft Check-in (Day 3), Sales Push (Day 7), Trial Expiry (3 days before). | From stage sticky notes |

