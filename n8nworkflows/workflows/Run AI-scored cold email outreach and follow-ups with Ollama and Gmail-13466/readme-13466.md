Run AI-scored cold email outreach and follow-ups with Ollama and Gmail

https://n8nworkflows.xyz/workflows/run-ai-scored-cold-email-outreach-and-follow-ups-with-ollama-and-gmail-13466


# Run AI-scored cold email outreach and follow-ups with Ollama and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title (given):** Run AI-scored cold email outreach and follow-ups with Ollama and Gmail  
**Workflow internal name:** AI SDR — Fully Automated Cold Outreach with Scoring and Follow-ups

This workflow automates an outbound cold email sequence driven by a Google Sheet: it ingests new leads, scrapes their websites, uses an LLM (Ollama via LangChain nodes) to score lead fit and generate personalized emails, sends the initial email via Gmail, schedules follow-ups, detects replies in Gmail, and updates the sheet + sends Telegram notifications.

### 1.1 Lead Intake & Validation (Weekdays 8 AM)
- Reads all rows from Google Sheets.
- Filters leads with `Status = "new"` and required fields.
- Branches into “process leads” vs “no new leads”.

### 1.2 Research → Dossier → AI Scoring
- Scrapes the lead website (best-effort).
- Builds a dossier string.
- Calls LLM to return structured JSON scoring (0–100).
- Extracts score robustly (handles malformed model outputs).

### 1.3 Decision: Pursue vs Skip
- If score >= 40, proceeds to email generation/sending.
- Otherwise marks as `low_fit_skipped` and notifies Telegram.

### 1.4 Email Generation → Formatting → Send Email 1
- Calls LLM to generate 3 emails in strict JSON format.
- Extracts/normalizes subjects and bodies, produces HTML.
- Sends Email 1 via Gmail.
- Writes the generated content + status timestamps back to the sheet.
- Sends Telegram notification.

### 1.5 Follow-up Engine (Every 2 hours on weekdays)
- Scans the sheet to determine follow-up actions:
  - Email 2 after 3 days from Email 1 sent date
  - Email 3 after 7 days from Email 1 sent date (and only if Email 2 already sent)
- Sends follow-ups via Gmail and updates the sheet.
- Telegram notifications for each follow-up.

### 1.6 Reply Detection (Every 2 hours on weekdays)
- Filters active leads (in sequence) from the sheet.
- Searches Gmail for messages from the lead since Email 1 sent date, excluding your own sent messages.
- If a reply is found: marks lead as `replied`, stores reply metadata, and alerts Telegram.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / On-canvas Notes (Non-executing)
**Overview:** Sticky notes provide setup instructions, warnings, and a functional map of the workflow.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`

#### Node: Sticky Note
- **Type/role:** Sticky Note; describes entire system and setup/customization.
- **Key content:** Required columns, credentials, score threshold, cron timing, and where to edit sender name/Gmail address in code nodes.
- **Failure/edge cases:** None (non-executing).

#### Node: Sticky Note2 / Sticky Note3 / Sticky Note4
- **Type/role:** Sticky Notes; label the three main engines (research/score/send, follow-ups, reply detection).
- **Failure/edge cases:** None.

#### Node: Sticky Note1
- **Type/role:** Sticky Note warning; tells you to replace placeholder Gmail address in two code nodes to prevent self-detection as replies.
- **Failure/edge cases:** If not followed, reply detection will produce false positives.

---

### Block B — New Lead Processing Trigger → Read Sheet → Filter (Weekdays 8 AM)
**Overview:** Runs once each weekday morning, reads the sheet, and extracts leads marked as `new`.  
**Nodes involved:** `⏰ 8 AM — New Leads`, `📋 Read Sheet (New)`, `🆕 Filter New`, `🆕 Has Leads?`, `💤 No New Leads`

#### Node: ⏰ 8 AM — New Leads
- **Type/role:** Schedule Trigger; starts the “new lead” pipeline.
- **Config:** Cron `0 8 * * 1-5` (8:00 Mon–Fri).
- **Output:** Triggers execution.
- **Edge cases:** Server timezone affects actual run time.

#### Node: 📋 Read Sheet (New)
- **Type/role:** Google Sheets; reads lead rows.
- **Config choices:**
  - Document ID: `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - Sheet: `YOUR_SHEET_GID` / “Leads” (must be replaced/selected)
- **Output:** Items representing rows.
- **Failure modes:** OAuth/permissions; wrong sheet ID/gid; column name mismatches.

#### Node: 🆕 Filter New
- **Type/role:** Code; filters rows to actionable lead objects.
- **Logic highlights:**
  - Reads `Status` (case-insensitive) and requires `Lead Name` + `Email`.
  - Creates a normalized payload: `leadName, company, website, role, email, linkedinUrl, rowNumber`.
  - `rowNumber = i + 2` assumes row 1 is header.
  - If no leads: returns one item `{ hasNewLeads: false }`.
- **Connections:** Input from `📋 Read Sheet (New)` → Output to `🆕 Has Leads?`.
- **Edge cases:**
  - If sheet uses different headers, lead extraction fails silently (lead won’t pass filters).
  - Multiple “new” leads yield multiple items; downstream nodes run per-item.

#### Node: 🆕 Has Leads?
- **Type/role:** IF; determines whether `leadName` exists.
- **Condition:** `$json.leadName` is not empty.
- **True path:** `🌐 Scrape Website`
- **False path:** `💤 No New Leads`
- **Edge cases:** When `🆕 Filter New` returns `{hasNewLeads:false}` it will go false path.

#### Node: 💤 No New Leads
- **Type/role:** Code; produces an informational “no work” output.
- **Output:** `{ status: 'no_new_leads', checkedAt: <local string> }`
- **Edge cases:** None (informational).

---

### Block C — Website Research & Dossier Assembly
**Overview:** Attempts to fetch and summarize the lead’s website and builds a dossier string used by the LLM.  
**Nodes involved:** `🌐 Scrape Website`, `📄 Build Dossier`

#### Node: 🌐 Scrape Website
- **Type/role:** Code; HTTP scrape and text extraction.
- **Configuration/choices:**
  - Ensures URL has `https://` if missing.
  - Uses `this.helpers.httpRequest` with:
    - `timeout: 15000`
    - `User-Agent: Mozilla... research-bot`
    - `ignoreHttpStatusErrors: true`
    - `returnFullResponse: true`
  - Extracts `<title>` and meta description.
  - Strips scripts/styles/nav/footer/tags, normalizes whitespace, truncates content to 1500 chars.
  - If no website: writes a fallback research string.
- **Output:** Adds `research: {companyInfo, scrapedSuccessfully}`.
- **Edge cases/failure modes:**
  - Websites blocking bots, Cloudflare, redirects, or long pages.
  - Non-HTML bodies (JSON, PDF) become stringified noise.
  - Timeout (15s) → fallback “Could not scrape …”.

#### Node: 📄 Build Dossier
- **Type/role:** Code; constructs a dossier text block for the LLM.
- **Logic:** Creates `dossier` with person, role, company, website, email, research summary, scraped flag.
- **Output:** Adds `dossier` to lead item.

---

### Block D — AI Scoring (Ollama via LangChain Agent) + Robust Score Extraction
**Overview:** Scores lead fit using an LLM and converts varied model outputs into a reliable numeric score and a boolean pursue decision.  
**Nodes involved:** `📊 AI Lead Scorer`, `Ollama Scorer`, `🎯 Extract Score`, `📊 Pursue?`

#### Node: Ollama Scorer
- **Type/role:** LangChain Chat Model (Ollama); provides the LLM to the agent.
- **Config:** `model: YOUR_MODEL_NAME`, `temperature: 0.3`.
- **Connections:** Connected to `📊 AI Lead Scorer` via `ai_languageModel`.
- **Failure modes:** Ollama not reachable, wrong model name, model not pulled, latency/timeouts.

#### Node: 📊 AI Lead Scorer
- **Type/role:** LangChain Agent; prompts the model to output *raw JSON* scoring.
- **Prompt:** “Score this lead… {{ dossier }} … Score 0-100.”
- **System message:** Requires you to edit product/ICP/scoring; demands a specific JSON schema and “RAW JSON ONLY”.
- **Output:** Agent output which may be nested depending on node version.
- **Edge cases:**
  - LLM returns markdown fences or extra text; handled later.
  - If the agent’s output parser behavior changes between versions, extraction may need adjustment.
- **Version notes:** `@n8n/n8n-nodes-langchain.agent` v2.1; requires n8n with LangChain nodes installed/enabled.

#### Node: 🎯 Extract Score
- **Type/role:** Code; robust JSON extraction + defaulting.
- **Key logic:**
  - Uses lead from `$('📄 Build Dossier').item.json` (not from immediate input).
  - Tries to locate/parse JSON containing `"totalScore"` in multiple nesting keys (`output`, `text`, `message`, `content`, etc.) and via regex fallback.
  - If extraction fails: defaults to `totalScore: 50` with a warning assessment.
  - Computes:
    - `score`
    - `recommendation`
    - `shouldPursue: score >= 40` (threshold is here)
- **Edge cases:**
  - If `📄 Build Dossier` item indexing differs (multiple parallel branches), cross-node reference could mismatch. In this workflow, it stays linear for each lead.
  - Default score=50 can cause unintended emailing if model output is unparseable; consider defaulting lower or requiring manual review.

#### Node: 📊 Pursue?
- **Type/role:** IF; gates outreach based on `shouldPursue`.
- **Condition:** `$json.shouldPursue` is true.
- **True path:** `✍️ AI Email Writer`
- **False path:** `⏭️ Mark Skipped`

---

### Block E — Email Generation (LLM) → Extract/Normalize → Send Email 1 → Persist & Notify
**Overview:** Creates 3 personalized emails with AI, normalizes formatting, sends Email 1, writes all email content to the sheet, and notifies Telegram.  
**Nodes involved:** `✍️ AI Email Writer`, `Ollama Writer`, `📝 Extract Emails`, `📧 Send Email 1`, `💾 Save + Mark Sent`, `📲 Email 1 Sent ✅`

#### Node: Ollama Writer
- **Type/role:** LangChain Chat Model (Ollama).
- **Config:** `model: YOUR_MODEL_NAME`, `temperature: 0.7`.
- **Connections:** Feeds `✍️ AI Email Writer` via `ai_languageModel`.

#### Node: ✍️ AI Email Writer
- **Type/role:** LangChain Agent; generates 3 emails as raw JSON.
- **System message:** Contains strict formatting and content rules; requires editing sender info placeholders.
- **Output format:** JSON with `email1/email2/email3 {subject, body}` and `personalizationUsed`.
- **Edge cases:** LLM may violate formatting rules; handled partially by extraction code.

#### Node: 📝 Extract Emails
- **Type/role:** Code; extracts JSON emails robustly, formats subject/body, creates HTML, sets reply-style subject prefixes.
- **Critical configuration:** You must set:
  - `const senderName = 'YOUR_NAME';` (line noted in sticky note)
- **Key behaviors:**
  - `firstName` = first token of `leadName`
  - Strips markdown artifacts and “Subject:” lines.
  - Ensures greeting exists and appends standardized sign-off.
  - Adds `Re:` to subjects for Email 2/3 (thread-like behavior).
  - Builds HTML wrappers.
  - If parsing fails: uses fallback email templates.
- **Outputs:** `email1Subject`, `email1Body`, `email1HTML`, similarly for 2/3, plus `email1Full/2Full/3Full`.
- **Edge cases:**
  - Over-aggressive signature stripping could remove intended content.
  - If `lead.leadName` is empty/unusual, greeting could be odd.
  - HTML conversion is simplistic (paragraph splitting on blank lines).

#### Node: 📧 Send Email 1
- **Type/role:** Gmail; sends the first email.
- **Config:**
  - To: `{{$json.email}}`
  - Subject: `{{$json.email1Subject}}`
  - Message: `{{$json.email1HTML}}`
  - `appendAttribution: false`
- **Failure modes:** Gmail OAuth expiry; Gmail sending limits; invalid recipient; Google policy restrictions.

#### Node: 💾 Save + Mark Sent
- **Type/role:** Google Sheets appendOrUpdate; persists sequence data and marks status.
- **Matching key:** `Lead Name`
- **Writes:**
  - `Status = email_1_sent`
  - `email 1 sent date = now ISO`
  - Saves subjects/bodies/html for emails 1–3
- **Important:** Column names must match exactly (note: workflow uses both `email 1subject` and `email 2 subject` with specific spacing).
- **Edge cases:**
  - Duplicate lead names will overwrite the wrong row (because matching is by Lead Name only).
  - If sheet headers differ (extra spaces), updates may silently fail or create new columns.

#### Node: 📲 Email 1 Sent ✅
- **Type/role:** Telegram sendMessage; notifies about Email 1.
- **Config:** `chatId: YOUR_TELEGRAM_CHAT_ID`, `parse_mode: Markdown`.
- **Uses expressions:** Pulls fields from `📝 Extract Emails`.
- **Failure modes:** Wrong chat ID; bot not started by user; Markdown formatting issues.

---

### Block F — Low-fit Skip Path
**Overview:** For leads below the threshold, marks them skipped and sends a Telegram notice.  
**Nodes involved:** `⏭️ Mark Skipped`, `📲 Skipped`

#### Node: ⏭️ Mark Skipped
- **Type/role:** Google Sheets appendOrUpdate.
- **Matching key:** `Lead Name`
- **Writes:** `Status = low_fit_skipped`
- **Edge cases:** Same duplicate-name overwrite risk.

#### Node: 📲 Skipped
- **Type/role:** Telegram sendMessage; notifies skip.
- **Config:** Markdown parse mode.
- **Failure modes:** Same as other Telegram nodes.

---

### Block G — Follow-up Engine (Every 2 hours weekdays)
**Overview:** Determines whether Email 2 or Email 3 should be sent based on time since Email 1 and status columns, then sends and updates the sheet.  
**Nodes involved:** `⏰ Every 2hrs — Follow-ups`, `📋 Read Sheet (Follow-ups)`, `🔍 Find Follow-ups`, `📧 Has Follow-ups?`, `📧 Send Follow-up`, `📋 Prepare Update`, `✅ Mark Follow-up Sent`, `📲 Follow-up Sent`, `💤 Nothing Pending`

#### Node: ⏰ Every 2hrs — Follow-ups
- **Type/role:** Schedule Trigger.
- **Cron:** `0 10,12,14,16 * * 1-5`
- **Meaning:** Runs at 10:00,12:00,14:00,16:00 Mon–Fri.

#### Node: 📋 Read Sheet (Follow-ups)
- **Type/role:** Google Sheets; reads all leads.

#### Node: 🔍 Find Follow-ups
- **Type/role:** Code; scans rows and emits “actions” to send follow-ups.
- **Critical configuration:** `const senderName = 'YOUR_NAME';` must be edited.
- **Selection logic:**
  - Skips statuses: `low_fit_skipped`, `sequence_complete`, `replied`, `new`
  - Requires `email 1 sent date`
  - Computes `daysSinceEmail1`
  - If Email 2 not sent and >=3 days → action `send_email_2`
  - Else if Email 2 sent and Email 3 not sent and >=7 days → action `send_email_3`
- **Subject/body sources:** Pulls `email 2 subject`, `email 2 body`, etc., with fallbacks.
- **Notable edge:** It reads Email 2 sent date from column `email 2 sent date ` (with a trailing space) OR `email 2 sent date`. The update later writes to `email 2 sent date ` (with trailing space). Your sheet header must match one of these.
- **Output:** Either actions list, or a single `{hasActions:false,...}` item.

#### Node: 📧 Has Follow-ups?
- **Type/role:** IF; checks `$json.action` not empty.
- **True path:** send follow-up
- **False path:** `💤 Nothing Pending`

#### Node: 📧 Send Follow-up
- **Type/role:** Gmail send.
- **Inputs:** `to=$json.email`, `subject=$json.subject`, `message=$json.htmlBody`.
- **Failure modes:** Gmail limits, invalid email, auth.

#### Node: 📋 Prepare Update
- **Type/role:** Code; prepares the exact columns to update.
- **Logic:**
  - If `emailNumber === 2`:
    - `Status=email_2_sent`
    - `email 2 sent date ` = now ISO (note trailing space)
    - sets subject/body/html
  - If `emailNumber === 3`:
    - `Status=sequence_complete`
    - `email 3 sent date` = now ISO
    - sets subject/body/html
- **Edge cases:** If `📧 Has Follow-ups?` item doesn’t exist (unexpected branching), cross-node reference can break.

#### Node: ✅ Mark Follow-up Sent
- **Type/role:** Google Sheets appendOrUpdate.
- **Matching key:** `Lead Name`
- **Writes:** status and appropriate timestamp + email content.

#### Node: 📲 Follow-up Sent
- **Type/role:** Telegram sendMessage.
- **Text:** Includes which follow-up number and whether sequence completed.

#### Node: 💤 Nothing Pending
- **Type/role:** Code; informational output if no actions.

---

### Block H — Reply Detection (Every 2 hours weekdays)
**Overview:** For leads with statuses indicating an active/finished sequence, searches Gmail inbox for messages from them after Email 1 date, excludes your own address, then marks replied and notifies.  
**Nodes involved:** `⏰ Every 2hrs — Reply Check`, `📋 Read Sheet (Replies)`, `🔍 Filter Active Leads`, `📬 Has Active Leads?`, `🔎 Search Gmail Replies`, `📩 Check Reply Results`, `💬 Reply Found?`, `✅ Mark Replied in Sheet`, `📲 Reply Alert! 🎉`, `💤 No Reply Yet`, `💤 No Active Leads`

#### Node: ⏰ Every 2hrs — Reply Check
- **Type/role:** Schedule Trigger.
- **Cron:** `30 9,11,13,15,17 * * 1-5` (09:30,11:30,13:30,15:30,17:30 weekdays)

#### Node: 📋 Read Sheet (Replies)
- **Type/role:** Google Sheets; reads all leads.

#### Node: 🔍 Filter Active Leads
- **Type/role:** Code; selects leads to check.
- **Critical configuration:** Replace:
  - `const MY_GMAIL = 'user@example.com';` with your sending Gmail.
- **Logic:**
  - Active statuses: `email_1_sent`, `email_2_sent`, `sequence_complete`
  - Requires `email 1 sent date`
  - Emits `{leadName,email,company,status,email1SentDate,myGmail,rowIndex}`
- **Edge cases:** If statuses are inconsistent (typos/case), leads are skipped.

#### Node: 📬 Has Active Leads?
- **Type/role:** IF; checks `$json.email` not empty.
- **True path:** Gmail search
- **False path:** `💤 No Active Leads`

#### Node: 🔎 Search Gmail Replies
- **Type/role:** Gmail getAll.
- **Config:**
  - `q = from:<lead email> in:inbox newer_than:30d -in:sent`
  - `limit: 30`
  - `simple: false` (returns richer structure)
- **Edge cases:**
  - Gmail search syntax limitations; aliases; “from” mismatch if sender differs.
  - Replies older than 30 days won’t be detected.
  - Threads where reply comes from different address/domain won’t match.

#### Node: 📩 Check Reply Results
- **Type/role:** Code; determines whether any fetched message is a real reply.
- **Critical configuration:** Ensures `MY_GMAIL` comes from lead.myGmail, defaulting to placeholder; must be set correctly.
- **Core logic:**
  - Parses Email 1 sent date robustly (supports ISO, epoch ms/s, numeric).
  - Iterates messages and skips:
    - messages labeled SENT (unless in INBOX)
    - messages from your own Gmail
    - messages whose sender doesn’t match lead email
    - messages dated before/at email1SentDate
  - Extracts `replyDate`, `replySubject`, `replySnippet` (up to 300 chars)
- **Edge cases:**
  - Gmail node message fields can vary by n8n version/config; code tries multiple keys.
  - Some inbound messages may lack parsed `from`/`internalDate`.
  - If Email 1 sent date in sheet is malformed, replies will never be detected (node returns debug info).

#### Node: 💬 Reply Found?
- **Type/role:** IF; checks boolean `$json.hasReply`.
- **True path:** mark replied + telegram alert (both in parallel)
- **False path:** `💤 No Reply Yet`

#### Node: ✅ Mark Replied in Sheet
- **Type/role:** Google Sheets appendOrUpdate.
- **Matching key:** `Lead Name`
- **Writes:** `Status=replied`, plus `Reply Date/Subject/Snippet`.
- **Edge cases:** Duplicate name overwrite risk.

#### Node: 📲 Reply Alert! 🎉
- **Type/role:** Telegram sendMessage with Markdown; alerts with snippet and stops sequence manually (sequence is “stopped” by status update).

#### Node: 💤 No Reply Yet / 💤 No Active Leads
- **Type/role:** Code informational outputs.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Global description + setup checklist | — | — | ## AI SDR — Fully Automated Cold Outreach with Scoring and Follow-ups… (full note content includes setup/customization and which lines to edit) |
| Sticky Note2 | Sticky Note | Labels research/score/send block | — | — | ## 🔬 Research, Score & Send… |
| Sticky Note3 | Sticky Note | Labels follow-up engine block | — | — | ## 📧 Follow-up Engine… |
| Sticky Note4 | Sticky Note | Labels reply detection block | — | — | ## 📬 Reply Detection… |
| Sticky Note1 | Sticky Note | Warning to set Gmail address in code nodes | — | — | ## ⚠️ Set your Gmail address… |
| ⏰ 8 AM — New Leads | Schedule Trigger | Starts new-lead processing | — | 📋 Read Sheet (New) |  |
| 📋 Read Sheet (New) | Google Sheets | Read all leads for “new” processing | ⏰ 8 AM — New Leads | 🆕 Filter New |  |
| 🆕 Filter New | Code | Extract rows with Status=new and normalize fields | 📋 Read Sheet (New) | 🆕 Has Leads? |  |
| 🆕 Has Leads? | IF | Branch: has a lead vs no new leads | 🆕 Filter New | 🌐 Scrape Website; 💤 No New Leads |  |
| 🌐 Scrape Website | Code | Scrape website and extract text summary | 🆕 Has Leads? | 📄 Build Dossier | ## 🔬 Research, Score & Send… |
| 📄 Build Dossier | Code | Build dossier string for LLM | 🌐 Scrape Website | 📊 AI Lead Scorer | ## 🔬 Research, Score & Send… |
| Ollama Scorer | Ollama Chat Model (LangChain) | LLM provider for scoring agent | — | (AI input to) 📊 AI Lead Scorer | ## 🔬 Research, Score & Send… |
| 📊 AI Lead Scorer | LangChain Agent | Produces lead score JSON | 📄 Build Dossier | 🎯 Extract Score | ## 🔬 Research, Score & Send… |
| 🎯 Extract Score | Code | Parse model output → score + shouldPursue | 📊 AI Lead Scorer | 📊 Pursue? | ## 🔬 Research, Score & Send… |
| 📊 Pursue? | IF | Gate outreach by score threshold | 🎯 Extract Score | ✍️ AI Email Writer; ⏭️ Mark Skipped | ## 🔬 Research, Score & Send… |
| Ollama Writer | Ollama Chat Model (LangChain) | LLM provider for email writer agent | — | (AI input to) ✍️ AI Email Writer | ## 🔬 Research, Score & Send… |
| ✍️ AI Email Writer | LangChain Agent | Generates 3 emails JSON | 📊 Pursue? | 📝 Extract Emails | ## 🔬 Research, Score & Send… |
| 📝 Extract Emails | Code | Parse/clean emails, add Re:, make HTML | ✍️ AI Email Writer | 📧 Send Email 1 | ## 🔬 Research, Score & Send… |
| 📧 Send Email 1 | Gmail | Sends first email | 📝 Extract Emails | 💾 Save + Mark Sent; 📲 Email 1 Sent ✅ | ## 🔬 Research, Score & Send… |
| 💾 Save + Mark Sent | Google Sheets | Save emails + set email_1_sent + timestamps | 📧 Send Email 1 | — | ## 🔬 Research, Score & Send… |
| 📲 Email 1 Sent ✅ | Telegram | Notify Email 1 sent | 📧 Send Email 1 | — | ## 🔬 Research, Score & Send… |
| ⏭️ Mark Skipped | Google Sheets | Mark low fit as skipped | 📊 Pursue? | 📲 Skipped | ## 🔬 Research, Score & Send… |
| 📲 Skipped | Telegram | Notify skipped lead | ⏭️ Mark Skipped | — | ## 🔬 Research, Score & Send… |
| 💤 No New Leads | Code | Informational output | 🆕 Has Leads? | — |  |
| ⏰ Every 2hrs — Follow-ups | Schedule Trigger | Starts follow-up scanning | — | 📋 Read Sheet (Follow-ups) | ## 📧 Follow-up Engine… |
| 📋 Read Sheet (Follow-ups) | Google Sheets | Read all leads for follow-up decisions | ⏰ Every 2hrs — Follow-ups | 🔍 Find Follow-ups | ## 📧 Follow-up Engine… |
| 🔍 Find Follow-ups | Code | Decide whether to send Email 2/3 | 📋 Read Sheet (Follow-ups) | 📧 Has Follow-ups? | ## 📧 Follow-up Engine… |
| 📧 Has Follow-ups? | IF | Branch: action exists or none | 🔍 Find Follow-ups | 📧 Send Follow-up; 💤 Nothing Pending | ## 📧 Follow-up Engine… |
| 📧 Send Follow-up | Gmail | Sends Email 2 or 3 | 📧 Has Follow-ups? | 📋 Prepare Update; 📲 Follow-up Sent | ## 📧 Follow-up Engine… |
| 📋 Prepare Update | Code | Build sheet update payload for follow-up | 📧 Send Follow-up | ✅ Mark Follow-up Sent | ## 📧 Follow-up Engine… |
| ✅ Mark Follow-up Sent | Google Sheets | Persist follow-up status + timestamps | 📋 Prepare Update | — | ## 📧 Follow-up Engine… |
| 📲 Follow-up Sent | Telegram | Notify follow-up sent | 📧 Send Follow-up | — | ## 📧 Follow-up Engine… |
| 💤 Nothing Pending | Code | Informational output | 📧 Has Follow-ups? | — | ## 📧 Follow-up Engine… |
| ⏰ Every 2hrs — Reply Check | Schedule Trigger | Starts reply detection scanning | — | 📋 Read Sheet (Replies) | ## 📬 Reply Detection… |
| 📋 Read Sheet (Replies) | Google Sheets | Read all leads for reply detection | ⏰ Every 2hrs — Reply Check | 🔍 Filter Active Leads | ## 📬 Reply Detection… |
| 🔍 Filter Active Leads | Code | Select leads with active statuses | 📋 Read Sheet (Replies) | 📬 Has Active Leads? | ## 📬 Reply Detection… / ## ⚠️ Set your Gmail address… |
| 📬 Has Active Leads? | IF | Branch: has active leads vs none | 🔍 Filter Active Leads | 🔎 Search Gmail Replies; 💤 No Active Leads | ## 📬 Reply Detection… |
| 🔎 Search Gmail Replies | Gmail | Search inbox for replies from lead | 📬 Has Active Leads? | 📩 Check Reply Results | ## 📬 Reply Detection… |
| 📩 Check Reply Results | Code | Validate reply found (exclude self, compare dates) | 🔎 Search Gmail Replies | 💬 Reply Found? | ## 📬 Reply Detection… / ## ⚠️ Set your Gmail address… |
| 💬 Reply Found? | IF | Branch: reply found vs not | 📩 Check Reply Results | ✅ Mark Replied in Sheet + 📲 Reply Alert! 🎉; 💤 No Reply Yet | ## 📬 Reply Detection… |
| ✅ Mark Replied in Sheet | Google Sheets | Mark lead replied + store reply metadata | 💬 Reply Found? | — | ## 📬 Reply Detection… |
| 📲 Reply Alert! 🎉 | Telegram | Notify reply detected | 💬 Reply Found? | — | ## 📬 Reply Detection… |
| 💤 No Reply Yet | Code | Informational output | 💬 Reply Found? | — | ## 📬 Reply Detection… |
| 💤 No Active Leads | Code | Informational output | 📬 Has Active Leads? | — | ## 📬 Reply Detection… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Google Sheets OAuth2 credential (access to the target spreadsheet).
   2. Gmail OAuth2 credential (the sending mailbox).
   3. Telegram credential (bot token).
   4. Ollama / LangChain Ollama credential or configuration (ensure Ollama reachable; model pulled).

2) **Prepare Google Sheet**
   - Create a sheet (tab) named “Leads” (or any, but pick it in nodes).
   - Ensure headers exist (case/spacing matters because code references them), including at least:
     - `Lead Name`, `Email`, `Company`, `Website`, `Role/Title`, `Status`
     - `email 1 sent date`, `email 1subject`, `email 1 body`, `email 1 body html`
     - `email 2 sent date ` (note trailing space) and/or `email 2 sent date`
     - `email 2 subject`, `email 2 body`, `email 2 body html`
     - `email 3 sent date`, `email 3 sub`, `email 3 body`, `email 3 body html`
     - `Reply Date`, `Reply Subject`, `Reply Snippet`
   - Use `Status = new` for leads to be processed at 8 AM.

3) **Block: New lead trigger & read**
   1. Add **Schedule Trigger** named `⏰ 8 AM — New Leads`
      - Cron: `0 8 * * 1-5`
   2. Add **Google Sheets** node named `📋 Read Sheet (New)`
      - Operation: read/get many rows (default “Read” behavior)
      - Select Document ID and Sheet/Tab (“Leads”)
   3. Connect trigger → read node.

4) **Filter new leads**
   1. Add **Code** node `🆕 Filter New` with the provided logic concept:
      - Filter rows where `Status` equals `new` (case-insensitive)
      - Require lead name and email
      - Output normalized fields + `rowNumber = i + 2`
      - If none, output `{hasNewLeads:false}`
   2. Connect `📋 Read Sheet (New)` → `🆕 Filter New`.
   3. Add **IF** node `🆕 Has Leads?`
      - Condition: `{{$json.leadName}}` is not empty
   4. Connect `🆕 Filter New` → `🆕 Has Leads?`
   5. Add **Code** node `💤 No New Leads` that outputs a status message.
   6. Connect `🆕 Has Leads?` false output → `💤 No New Leads`.

5) **Research + dossier**
   1. Add **Code** node `🌐 Scrape Website`
      - Implement: if no website → fallback summary
      - Else http GET with 15s timeout, strip HTML, extract title/meta description, truncate
   2. Add **Code** node `📄 Build Dossier` to format a dossier string.
   3. Connect `🆕 Has Leads?` true output → `🌐 Scrape Website` → `📄 Build Dossier`.

6) **AI scoring (Ollama + agent)**
   1. Add **Ollama Chat Model** node `Ollama Scorer`
      - Model: your local model name (e.g., `llama3.1`, `mistral`, etc.)
      - Temperature: 0.3
   2. Add **LangChain Agent** node `📊 AI Lead Scorer`
      - Prompt includes `{{ $json.dossier }}` and requests “Score 0-100.”
      - System message: paste and edit “YOUR PRODUCT / ICP / scoring” section.
      - Enforce “RAW JSON ONLY” schema.
   3. Connect `📄 Build Dossier` → `📊 AI Lead Scorer` (main)
   4. Connect `Ollama Scorer` → `📊 AI Lead Scorer` (AI Language Model connection).

7) **Extract score + gate**
   1. Add **Code** node `🎯 Extract Score`
      - Parse nested agent outputs to retrieve JSON with `totalScore`
      - Compute `shouldPursue = totalScore >= 40`
      - (Optional improvement) change default score behavior if parse fails
   2. Add **IF** node `📊 Pursue?`
      - Condition: `{{$json.shouldPursue}}` is true
   3. Connect `📊 AI Lead Scorer` → `🎯 Extract Score` → `📊 Pursue?`.

8) **Skip path**
   1. Add **Google Sheets** node `⏭️ Mark Skipped`
      - Operation: Append or Update
      - Match column: `Lead Name`
      - Set `Status = low_fit_skipped`
   2. Add **Telegram** node `📲 Skipped`
      - chatId: your chat id
      - parse_mode: Markdown
      - Message uses lead fields (name/company/score/assessment)
   3. Connect `📊 Pursue?` false output → `⏭️ Mark Skipped` → `📲 Skipped`.

9) **Email generation (Ollama + agent)**
   1. Add **Ollama Chat Model** node `Ollama Writer`
      - Model: same or different model
      - Temperature: 0.7
   2. Add **LangChain Agent** node `✍️ AI Email Writer`
      - Prompt references `{{ $json.dossier }}`, score, keyInsight, assessment
      - System message: paste strict formatting rules and edit sender info placeholders
      - Output format: raw JSON with email1/email2/email3 objects
   3. Connect `📊 Pursue?` true output → `✍️ AI Email Writer`.
   4. Connect `Ollama Writer` → `✍️ AI Email Writer` (AI Language Model connection).

10) **Extract/format emails**
   1. Add **Code** node `📝 Extract Emails`
      - **Set `senderName` to your real name**
      - Parse agent output robustly
      - Enforce greeting + sign-off, create HTML, add `Re:` for follow-ups
   2. Connect `✍️ AI Email Writer` → `📝 Extract Emails`.

11) **Send Email 1 + persist + notify**
   1. Add **Gmail** node `📧 Send Email 1`
      - Operation: Send
      - To: `{{$json.email}}`
      - Subject: `{{$json.email1Subject}}`
      - Message: `{{$json.email1HTML}}`
      - appendAttribution: false
   2. Add **Google Sheets** node `💾 Save + Mark Sent`
      - Operation: Append or Update
      - Match: `Lead Name`
      - Write: `Status=email_1_sent`, set `email 1 sent date` to now ISO, and save email bodies/HTML for 1–3
   3. Add **Telegram** node `📲 Email 1 Sent ✅`
      - chatId: your chat id
      - parse_mode: Markdown
      - Include score, subject, timing note
   4. Connect `📝 Extract Emails` → `📧 Send Email 1`.
   5. Connect `📧 Send Email 1` → `💾 Save + Mark Sent` and also to `📲 Email 1 Sent ✅` (parallel).

12) **Follow-up engine**
   1. Add **Schedule Trigger** `⏰ Every 2hrs — Follow-ups` with cron `0 10,12,14,16 * * 1-5`
   2. Add **Google Sheets** `📋 Read Sheet (Follow-ups)` (read all rows)
   3. Add **Code** `🔍 Find Follow-ups`
      - **Set `senderName`**
      - Implement 3-day and 7-day logic based on `email 1 sent date`
      - Respect skipped/replied/new statuses
   4. Add **IF** `📧 Has Follow-ups?` checking `$json.action` not empty
   5. Add **Gmail** `📧 Send Follow-up` using `{{$json.email}}`, `{{$json.subject}}`, `{{$json.htmlBody}}`
   6. Add **Code** `📋 Prepare Update` to create the update payload (note the `email 2 sent date ` trailing space if you keep it)
   7. Add **Google Sheets** `✅ Mark Follow-up Sent` appendOrUpdate matching `Lead Name`
   8. Add **Telegram** `📲 Follow-up Sent`
   9. Add **Code** `💤 Nothing Pending`
   10. Wire: Trigger → Read → Find → IF → (true) Send Follow-up → Prepare Update → Mark Sent; also Send Follow-up → Telegram. IF false → Nothing Pending.

13) **Reply detection engine**
   1. Add **Schedule Trigger** `⏰ Every 2hrs — Reply Check` cron `30 9,11,13,15,17 * * 1-5`
   2. Add **Google Sheets** `📋 Read Sheet (Replies)`
   3. Add **Code** `🔍 Filter Active Leads`
      - **Set `MY_GMAIL` to your sender Gmail address**
      - Only statuses: `email_1_sent`, `email_2_sent`, `sequence_complete`
   4. Add **IF** `📬 Has Active Leads?` (checks `$json.email` not empty)
   5. Add **Gmail** `🔎 Search Gmail Replies`
      - Operation: Get Many (getAll)
      - Query: `from:{{$json.email}} in:inbox newer_than:30d -in:sent`
      - limit 30, `simple=false`
   6. Add **Code** `📩 Check Reply Results`
      - **Ensure it excludes your own Gmail (`MY_GMAIL`)**
      - Only counts messages after `email 1 sent date`
      - Extracts reply date/subject/snippet
   7. Add **IF** `💬 Reply Found?` where `{{$json.hasReply}}` is true
   8. Add **Google Sheets** `✅ Mark Replied in Sheet` appendOrUpdate matching `Lead Name` and writing reply fields
   9. Add **Telegram** `📲 Reply Alert! 🎉`
   10. Add **Code** `💤 No Reply Yet`
   11. Add **Code** `💤 No Active Leads`
   12. Wire: Trigger → Read → Filter → IF → (true) Search Gmail → Check Results → IF → (true) Mark Replied + Telegram; (false) No Reply Yet. IF false → No Active Leads.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Connect Google Sheets, Gmail (OAuth2), Telegram, and Ollama credentials; edit prompts and sender identity; adjust scoring threshold and cron schedules. | From the main sticky note (“AI SDR — Fully Automated… Setup steps / Customization”). |
| Edit `senderName` in **📝 Extract Emails** and **🔍 Find Follow-ups** code nodes. | Mentioned in sticky note setup steps. |
| Replace placeholder Gmail address in **🔍 Filter Active Leads** and **📩 Check Reply Results** to avoid false reply detection. | Warning sticky note (“⚠️ Set your Gmail address”). |
| Column naming consistency is critical (notably `email 2 sent date ` has a trailing space in updates). | Observed in follow-up code + sheet update nodes. |