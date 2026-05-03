Nurture and email leads from Google Sheets with GPT‑4.1 Mini and Gmail

https://n8nworkflows.xyz/workflows/nurture-and-email-leads-from-google-sheets-with-gpt-4-1-mini-and-gmail-14355


# Nurture and email leads from Google Sheets with GPT‑4.1 Mini and Gmail

# 1. Workflow Overview

This workflow automates a lightweight outbound lead-nurturing pipeline using Google Sheets as a CRM, OpenAI GPT‑4.1 Mini for lead analysis and message generation, and Gmail for email delivery.

Its main purpose is to:
- periodically read leads from Google Sheets,
- identify leads that need outreach,
- normalize lead data,
- score and classify each lead with AI,
- send either an initial cold email or a follow-up email,
- update the sheet with the current outreach stage,
- and eventually close inactive leads if no response is recorded.

The workflow is organized into the following logical blocks:

## 1.1 Trigger and Lead Intake
A schedule-based trigger starts the workflow, reads rows from Google Sheets, and filters leads based on missing outreach state fields.

## 1.2 Data Preparation and Batch Processing
Lead fields are normalized into a common structure, then processed in batches to reduce pressure on downstream services.

## 1.3 AI Lead Analysis and Routing
Each lead is analyzed by GPT‑4.1 Mini to determine category, score, priority, and recommended outreach channel. The workflow then routes the lead according to its current CRM step.

## 1.4 First Outreach Automation
For new leads, the workflow generates a personalized first email, sends it via Gmail, and updates the Google Sheet with AI metadata and scheduling fields.

## 1.5 Follow-up and Closure Handling
For leads already at step 1, the workflow generates a follow-up message, sends it, updates the sheet, waits two days, and closes the lead if no response is present.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Lead Intake

### Overview
This block starts execution on a recurring schedule, retrieves rows from a Google Sheet, and filters for leads that appear eligible for processing. The current condition is broad: it passes rows where either `Status` is empty or `NextActionDate` is empty.

### Nodes Involved
- Schedule Trigger
- Get row(s) in sheet
- If

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger` — entry point that runs the workflow on a timed interval.
- **Configuration choices:** Configured to run every hour.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Get row(s) in sheet**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or failures:**
  - Workflow will not run unless activated.
  - High-frequency schedules can increase OpenAI/Gmail/Google Sheets usage unexpectedly.
- **Sub-workflow reference:** None.

#### Get row(s) in sheet
- **Type and role:** `n8n-nodes-base.googleSheets` — reads lead data from the target worksheet.
- **Configuration choices:** Uses a selected Google Sheets document and sheet tab (`gid=0`). No extra filtering options are configured in the node itself.
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Receives from **Schedule Trigger**; outputs to **If**.
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets credentials and access to the selected spreadsheet.
- **Edge cases or failures:**
  - Google authentication failure or expired credentials.
  - Spreadsheet/tab ID mismatch.
  - Empty sheet returns no useful items.
  - Column names used later must exist exactly as referenced.
- **Sub-workflow reference:** None.

#### If
- **Type and role:** `n8n-nodes-base.if` — filters incoming rows to determine whether they should continue.
- **Configuration choices:** Uses OR logic:
  - `Status` is empty, or
  - `NextActionDate` is empty.
- **Key expressions or variables used:**
  - `{{$json.Status}}`
  - `{{$json.NextActionDate}}`
- **Input and output connections:** Receives from **Get row(s) in sheet**; true branch goes to **Code in JavaScript**. No false branch is connected.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failures:**
  - This condition does **not** compare `NextActionDate` to the current date. It only checks emptiness.
  - Leads that already have a valid future `NextActionDate` but empty `Status` will still be processed.
  - Rows failing the condition are silently ignored because the false output is unused.
- **Sub-workflow reference:** None.

---

## 2.2 Data Preparation and Batch Processing

### Overview
This block standardizes lead fields into normalized properties used downstream, then iterates through the leads in batches of five. It is mainly intended to simplify prompt building and reduce API burst load.

### Nodes Involved
- Code in JavaScript
- Loop Over Items

### Node Details

#### Code in JavaScript
- **Type and role:** `n8n-nodes-base.code` — transforms sheet rows into a normalized lead schema.
- **Configuration choices:** Builds derived fields:
  - `Email` from `Work Email` or `Personal Email`
  - `Phone` from `Work Number` or `Personal Number`
  - `Company` from `Provider Name`
  - `Person` from `First Name`
- **Key expressions or variables used:** JavaScript accesses source row fields directly, such as:
  - `data["Work Email"]`
  - `data["Personal Email"]`
  - `data["Provider Name"]`
  - `data["First Name"]`
- **Input and output connections:** Receives from **If**; outputs to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failures:**
  - The code references `data["Personal Number"]`, but the sheet schema elsewhere shows `Personal Number ` with a trailing space. This mismatch can cause phone fallback failure.
  - If both email fields are missing, `Email` becomes an empty string, which will later break Gmail sending.
  - No validation is performed on email format.
- **Sub-workflow reference:** None.

#### Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches` — processes items in chunks.
- **Configuration choices:** Batch size is set to `5`.
- **Key expressions or variables used:** None in configuration.
- **Input and output connections:**
  - Receives from **Code in JavaScript**
  - Batch output goes to **AI-Lead-Analysis**
  - Continuation is fed by **Update row in sheet** and **Update row in sheet2**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or failures:**
  - Split-in-batches workflows require correct loop-back wiring; otherwise items can stall or not complete.
  - Because different branches loop back into the same batch node, testing should confirm all branches advance correctly.
- **Sub-workflow reference:** None.

---

## 2.3 AI Lead Analysis and Routing

### Overview
This block sends each normalized lead to GPT‑4.1 Mini for qualification, parses the structured result, and routes the lead into either first outreach or follow-up logic based on current CRM state.

### Nodes Involved
- AI-Lead-Analysis
- Parse AI
- Switch

### Node Details

#### AI-Lead-Analysis
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi` — uses OpenAI Responses-style generation to analyze a lead.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Prompt includes `Company`, `Website`, `Title`, and `Country`
  - System instruction asks for strict B2B sales evaluation
  - Output must be JSON only
- **Key expressions or variables used:**
  - `{{$json.Company}}`
  - `{{$json.Website}}`
  - `{{$json.Title}}`
  - `{{$json.Country}}`
- **Input and output connections:** Receives from **Loop Over Items**; outputs to **Parse AI**.
- **Version-specific requirements:** Type version `2.1`; requires OpenAI credentials and support for the selected model in the account/region.
- **Edge cases or failures:**
  - Model may still return malformed JSON despite instruction.
  - API quota/rate limit failures.
  - Missing fields reduce analysis quality.
- **Sub-workflow reference:** None.

#### Parse AI
- **Type and role:** `n8n-nodes-base.code` — extracts model text from the OpenAI response and parses JSON safely.
- **Configuration choices:**
  - Reads `output[0].content`
  - Looks for `output_text`
  - Falls back to first content element or `"{}"`
  - On parse failure, returns defaults
- **Key expressions or variables used:** JavaScript path parsing:
  - `data.output?.[0]?.content?.find(...)?.text`
- **Output fields created:**
  - `AI_Category`
  - `AI_LeadScore`
  - `AI_Priority`
  - `Channel`
  - `AI_Reason`
  - `AI_Status`
- **Input and output connections:** Receives from **AI-Lead-Analysis**; outputs to **Switch**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failures:**
  - If OpenAI response structure changes, extraction may fail.
  - Invalid JSON yields fallback values and `AI_Status = "fallback"`.
  - If `recommended_channel` is absent, defaults to `Email`.
- **Sub-workflow reference:** None.

#### Switch
- **Type and role:** `n8n-nodes-base.switch` — routes leads by lifecycle stage.
- **Configuration choices:** Two routing rules:
  1. `Status` is empty
  2. `Step` equals `1`
- **Key expressions or variables used:**
  - `{{ $('Loop Over Items').item.json.Status }}`
  - `{{ $('Loop Over Items').item.json.Step }}`
- **Input and output connections:**
  - Receives from **Parse AI**
  - Output 0 → **First AI mail**
  - Output 1 → **AI-Follow-up**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or failures:**
  - The first rule uses a **number-empty** operator against `Status`, which is conceptually a string field. This may behave unexpectedly.
  - There is no explicit default branch, so unmatched leads are dropped.
  - A lead with `Status` populated and `Step` not equal to `1` will not be processed further.
- **Sub-workflow reference:** None.

---

## 2.4 First Outreach Automation

### Overview
This block generates a personalized cold email for new leads, sends it via Gmail, and updates the CRM row with AI analysis, message content, and next follow-up date.

### Nodes Involved
- First AI mail
- Save Message
- Send a message
- Update row in sheet

### Node Details

#### First AI mail
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi` — generates the initial cold email.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Prompt asks for a highly personalized cold email
  - Constraints:
    - max 2 lines
    - no fluff
    - no AI tone
    - clear benefit
    - avoid generic intros
  - Output only the email body
- **Key expressions or variables used:**
  - `{{$json.Company}}`
  - `{{$json.Person}}`
  - `{{$json.Title}}`
- **Input and output connections:** Receives from **Switch** output 0; outputs to **Save Message**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or failures:**
  - Missing `Person`, `Company`, or `Title` results in generic content.
  - AI output may still include formatting or extra quotes.
- **Sub-workflow reference:** None.

#### Save Message
- **Type and role:** `n8n-nodes-base.code` — extracts generated email text and stores it in `AI_FirstMessage`.
- **Configuration choices:** Reads OpenAI output text with a fallback message:
  - `Hi {Person}, quick question regarding {Company}`
- **Key expressions or variables used:** Uses response extraction logic similar to **Parse AI**.
- **Input and output connections:** Receives from **First AI mail**; outputs to **Send a message**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failures:**
  - If OpenAI output shape changes, extraction can fail.
  - Fallback message may be awkward if `Person` or `Company` is empty.
- **Sub-workflow reference:** None.

#### Send a message
- **Type and role:** `n8n-nodes-base.gmail` — sends the first outreach email.
- **Configuration choices:**
  - Recipient: `{{$json.Email}}`
  - Message body: `{{$json.AI_FirstMessage}}`
  - Subject: `Quick question`
- **Key expressions or variables used:**
  - `{{$json.Email}}`
  - `{{$json.AI_FirstMessage}}`
- **Input and output connections:** Receives from **Save Message**; outputs to **Update row in sheet**.
- **Version-specific requirements:** Type version `2.2`; requires Gmail credentials with send permission.
- **Edge cases or failures:**
  - Empty or invalid recipient email.
  - Gmail sending limits or account restrictions.
  - OAuth reauthorization may be required.
- **Sub-workflow reference:** None.

#### Update row in sheet
- **Type and role:** `n8n-nodes-base.googleSheets` — updates the current lead after first contact.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `#`
  - Sets:
    - `Step = 1`
    - `Status = Contacted`
    - `Channel`
    - `AI_Category`
    - `AI_Priority`
    - `AI_LeadScore`
    - `LastContacted = now`
    - `NextActionDate = now + 2 days`
    - `AI_FirstMessage`
- **Key expressions or variables used:**
  - `{{ $('Get row(s) in sheet').item.json['#'] }}`
  - `{{$json.Channel}}`
  - `{{$json.AI_Category}}`
  - `{{$json.AI_Priority}}`
  - `{{$json.AI_LeadScore}}`
  - `{{new Date().toISOString()}}`
  - `{{new Date(Date.now() + 2*24*60*60*1000).toISOString()}}`
- **Input and output connections:** Receives from **Send a message**; loops back to **Loop Over Items**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failures:**
  - Matching by `#` requires that field to be unique and present.
  - Cross-node item referencing using `$('Get row(s) in sheet').item...` can become fragile when batching or branching changes.
  - Date is stored in ISO format; ensure sheet consumers expect that format.
- **Sub-workflow reference:** None.

---

## 2.5 Follow-up and Closure Handling

### Overview
This block handles second-touch outreach for leads at step 1. It generates a short follow-up, sends it, updates the CRM, waits two days, then checks for response absence and closes the lead if no reply has been recorded.

### Nodes Involved
- AI-Follow-up
- Parse Follow-up
- Send a message1
- Update row in sheet1
- Wait1
- If1
- Update row in sheet2

### Node Details

#### AI-Follow-up
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi` — generates a short follow-up email.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Prompt uses `Company`
  - Rules:
    - friendly
    - not pushy
    - max 2 lines
    - natural tone
  - Output only the email
- **Key expressions or variables used:**
  - `{{$json.Company}}`
- **Input and output connections:** Receives from **Switch** output 1; outputs to **Parse Follow-up**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or failures:**
  - Limited context may make message too generic.
  - API failures or malformed response content.
- **Sub-workflow reference:** None.

#### Parse Follow-up
- **Type and role:** `n8n-nodes-base.code` — extracts follow-up content into a dedicated field.
- **Configuration choices:** Saves generated text to `AI_FollowupMessage` with fallback:
  - `Just following up on my previous message 🙂`
- **Key expressions or variables used:** Similar OpenAI output extraction logic to other code nodes.
- **Input and output connections:** Receives from **AI-Follow-up**; outputs to **Send a message1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failures:**
  - Response parsing issues.
  - Fallback may be overly casual for some brands.
- **Sub-workflow reference:** None.

#### Send a message1
- **Type and role:** `n8n-nodes-base.gmail` — sends the follow-up email.
- **Configuration choices:**
  - Recipient: `{{$json.Email}}`
  - Message: `{{$json.AI_FollowupMessage}}`
  - Subject: `Follow-up: "Quick question"`
- **Key expressions or variables used:**
  - `{{$json.Email}}`
  - `{{$json.AI_FollowupMessage}}`
- **Input and output connections:** Receives from **Parse Follow-up**; outputs to **Update row in sheet1**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or failures:**
  - Empty email address.
  - Gmail quota/auth issues.
- **Sub-workflow reference:** None.

#### Update row in sheet1
- **Type and role:** `n8n-nodes-base.googleSheets` — updates row status after follow-up is sent.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `#`
  - Sets:
    - `Step = 2`
    - `Status = Follow-up`
    - `LastContacted = now`
    - `NextActionDate = now + 3 days`
    - `AI_FollowupMessage`
- **Key expressions or variables used:**
  - `{{ $('Get row(s) in sheet').item.json['#'] }}`
  - `{{new Date().toISOString()}}`
  - `{{new Date(Date.now() + 3*24*60*60*1000).toISOString()}}`
  - `{{$json.AI_FollowupMessage}}`
- **Input and output connections:** Receives from **Send a message1**; outputs to **Wait1**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failures:**
  - Same row-matching fragility as other sheet updates.
  - If the sheet is edited concurrently, the update may affect unintended data if `#` is not stable.
- **Sub-workflow reference:** None.

#### Wait1
- **Type and role:** `n8n-nodes-base.wait` — pauses execution for two days before checking response state.
- **Configuration choices:** Wait duration is 2 days.
- **Key expressions or variables used:** None.
- **Input and output connections:** Receives from **Update row in sheet1**; outputs to **If1**.
- **Version-specific requirements:** Type version `1.1`; requires n8n wait/resume capability in the deployment.
- **Edge cases or failures:**
  - Long waits depend on persistent execution storage.
  - Instance restarts or misconfigured execution persistence may affect resumed runs.
- **Sub-workflow reference:** None.

#### If1
- **Type and role:** `n8n-nodes-base.if` — checks whether a response has been recorded.
- **Configuration choices:** Continues only if `Response` is empty.
- **Key expressions or variables used:**
  - `{{ $('Loop Over Items').item.json.Response }}`
- **Input and output connections:** Receives from **Wait1**; true branch goes to **Update row in sheet2**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failures:**
  - It checks the in-memory item from before the wait, not necessarily a freshly re-read sheet row.
  - If someone manually updates `Response` in the sheet during the two-day wait, this expression may not reflect that change depending on execution data context.
  - False branch is unused.
- **Sub-workflow reference:** None.

#### Update row in sheet2
- **Type and role:** `n8n-nodes-base.googleSheets` — marks unreplied leads as closed.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `#`
  - Sets:
    - `Step = 3`
    - `Status = Closed`
    - `Response = "No replies from the lead till now. So the lead has been closed."`
- **Key expressions or variables used:**
  - `{{ $('Get row(s) in sheet').item.json['#'] }}`
- **Input and output connections:** Receives from **If1**; loops back to **Loop Over Items**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failures:**
  - The configured document ID and sheet values appear blank/incomplete in this node, so it may fail unless fixed after import.
  - Closure logic is based on absence of a response field, not on actual Gmail reply detection.
- **Sub-workflow reference:** None.

---

## 2.6 Documentation / Annotation Nodes

### Overview
These sticky notes are non-executable documentation elements embedded in the canvas. They describe the intended business flow and setup guidance.

### Nodes Involved
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11

### Node Details

#### Sticky Note7
- **Type and role:** `n8n-nodes-base.stickyNote` — top-level workflow description.
- **Configuration choices:** Contains purpose, audience, setup, and a sequencing tip.
- **Input and output connections:** None.
- **Edge cases or failures:** None; informational only.
- **Sub-workflow reference:** None.

#### Sticky Note8
- Describes Step 1: trigger and lead intake.

#### Sticky Note10
- Describes Step 2: data preparation and batching.

#### Sticky Note9
- Describes Step 3: AI lead scoring and routing.

#### Sticky Note11
- Describes Step 4: first outreach automation.

#### Sticky Note6
- Describes Step 5: follow-up and pipeline completion.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow every hour |  | Get row(s) in sheet | ## Step 1 – Trigger and lead intake<br>Runs on a schedule, fetches leads from Google Sheets, and filters only records that need outreach based on Status or NextActionDate. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Reads lead rows from the sheet | Schedule Trigger | If | ## Step 1 – Trigger and lead intake<br>Runs on a schedule, fetches leads from Google Sheets, and filters only records that need outreach based on Status or NextActionDate. |
| If | n8n-nodes-base.if | Filters rows where Status or NextActionDate is empty | Get row(s) in sheet | Code in JavaScript | ## Step 1 – Trigger and lead intake<br>Runs on a schedule, fetches leads from Google Sheets, and filters only records that need outreach based on Status or NextActionDate. |
| Code in JavaScript | n8n-nodes-base.code | Normalizes email, phone, company, and person fields | If | Loop Over Items | ## Step 2 – Data preparation and batching<br>Normalizes lead fields (email, phone, company, person) and processes leads in batches to ensure scalability and avoid rate limits. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Processes leads in batches of 5 | Code in JavaScript, Update row in sheet, Update row in sheet2 | AI-Lead-Analysis | ## Step 2 – Data preparation and batching<br>Normalizes lead fields (email, phone, company, person) and processes leads in batches to ensure scalability and avoid rate limits. |
| AI-Lead-Analysis | @n8n/n8n-nodes-langchain.openAi | Scores and classifies leads with OpenAI | Loop Over Items | Parse AI | ## Step 3 – AI lead scoring and routing<br>Analyzes each lead using OpenAI to assign score, priority, and channel, then routes leads based on outreach step (new, follow-up, closed). |
| Parse AI | n8n-nodes-base.code | Parses AI JSON output into workflow fields | AI-Lead-Analysis | Switch | ## Step 3 – AI lead scoring and routing<br>Analyzes each lead using OpenAI to assign score, priority, and channel, then routes leads based on outreach step (new, follow-up, closed). |
| Switch | n8n-nodes-base.switch | Routes leads to first email or follow-up branch | Parse AI | First AI mail, AI-Follow-up | ## Step 3 – AI lead scoring and routing<br>Analyzes each lead using OpenAI to assign score, priority, and channel, then routes leads based on outreach step (new, follow-up, closed). |
| First AI mail | @n8n/n8n-nodes-langchain.openAi | Generates the initial personalized cold email | Switch | Save Message | ## Step 4 – First outreach automation<br>Generates and sends a personalized cold email, then updates CRM with Step, Status, AI insights, and next action date. |
| Save Message | n8n-nodes-base.code | Saves initial email text into AI_FirstMessage | First AI mail | Send a message | ## Step 4 – First outreach automation<br>Generates and sends a personalized cold email, then updates CRM with Step, Status, AI insights, and next action date. |
| Send a message | n8n-nodes-base.gmail | Sends the first outreach email via Gmail | Save Message | Update row in sheet | ## Step 4 – First outreach automation<br>Generates and sends a personalized cold email, then updates CRM with Step, Status, AI insights, and next action date. |
| Update row in sheet | n8n-nodes-base.googleSheets | Updates CRM after first outreach | Send a message | Loop Over Items | ## Step 4 – First outreach automation<br>Generates and sends a personalized cold email, then updates CRM with Step, Status, AI insights, and next action date. |
| AI-Follow-up | @n8n/n8n-nodes-langchain.openAi | Generates follow-up email text | Switch | Parse Follow-up | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Parse Follow-up | n8n-nodes-base.code | Saves follow-up text into AI_FollowupMessage | AI-Follow-up | Send a message1 | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Send a message1 | n8n-nodes-base.gmail | Sends follow-up email via Gmail | Parse Follow-up | Update row in sheet1 | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Update row in sheet1 | n8n-nodes-base.googleSheets | Updates CRM after follow-up | Send a message1 | Wait1 | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Wait1 | n8n-nodes-base.wait | Delays closure check by 2 days | Update row in sheet1 | If1 | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| If1 | n8n-nodes-base.if | Checks whether Response is still empty | Wait1 | Update row in sheet2 | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Update row in sheet2 | n8n-nodes-base.googleSheets | Closes leads with no response | If1 | Loop Over Items | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for step 5 |  |  | ## Step 5 – Follow-up and pipeline completion<br>Creates and sends follow-up emails, updates CRM progression, and marks leads as closed once the outreach sequence is complete. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | ### AI lead nurturing and cold email automation with Google Sheets and OpenAI<br><br>Automate lead qualification, personalized outreach, and follow-ups using AI and a Google Sheets CRM.<br><br>**What it does**<br>• Reads leads from Google Sheets<br>• Uses AI to score and prioritize leads<br>• Sends personalized cold emails<br>• Generates follow-up messages automatically<br>• Tracks outreach status and next steps<br>• Updates CRM in real-time<br><br>**Who it’s for**<br>• B2B sales teams<br>• Agencies and freelancers<br>• Founders scaling outbound outreach<br><br>**Setup**<br>• Connect Google Sheets, OpenAI, Gmail<br>• Ensure columns: Email, Name, Company, Status, Step<br>• Do not hardcode credentials<br><br>**Tip**<br>Use Step + NextActionDate to control multi-step outreach sequences. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation for step 1 |  |  | ## Step 1 – Trigger and lead intake<br>Runs on a schedule, fetches leads from Google Sheets, and filters only records that need outreach based on Status or NextActionDate. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation for step 3 |  |  | ## Step 3 – AI lead scoring and routing<br>Analyzes each lead using OpenAI to assign score, priority, and channel, then routes leads based on outreach step (new, follow-up, closed). |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation for step 2 |  |  | ## Step 2 – Data preparation and batching<br>Normalizes lead fields (email, phone, company, person) and processes leads in batches to ensure scalability and avoid rate limits. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual documentation for step 4 |  |  | ## Step 4 – First outreach automation<br>Generates and sends a personalized cold email, then updates CRM with Step, Status, AI insights, and next action date. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Nurture and email leads from Google Sheets with GPT‑4.1 Mini and Gmail**

2. **Create a Schedule Trigger node**
   - Type: **Schedule Trigger**
   - Configure it to run every **1 hour**.
   - This is the workflow entry point.

3. **Create a Google Sheets node named `Get row(s) in sheet`**
   - Type: **Google Sheets**
   - Operation: get/read rows
   - Connect Google Sheets credentials.
   - Select the spreadsheet document.
   - Select the target sheet/tab, here shown as `gid=0`.
   - Connect: `Schedule Trigger → Get row(s) in sheet`

4. **Create an If node named `If`**
   - Type: **If**
   - Use **OR** logic.
   - Condition 1: `{{$json.Status}}` is empty
   - Condition 2: `{{$json.NextActionDate}}` is empty
   - Connect: `Get row(s) in sheet → If`
   - Use only the **true** output for the next step.

5. **Create a Code node named `Code in JavaScript`**
   - Type: **Code**
   - Language: JavaScript
   - Paste logic that maps each row to normalized fields:
     - `Email = Work Email || Personal Email || ''`
     - `Phone = Work Number || Personal Number || ''`
     - `Company = Provider Name || ''`
     - `Person = First Name || ''`
   - Keep original row data as well.
   - Connect: `If (true) → Code in JavaScript`

6. **Create a Split In Batches node named `Loop Over Items`**
   - Type: **Loop Over Items / Split In Batches**
   - Batch size: **5**
   - Connect: `Code in JavaScript → Loop Over Items`

7. **Create an OpenAI node named `AI-Lead-Analysis`**
   - Type: **OpenAI** (`@n8n/n8n-nodes-langchain.openAi`)
   - Credentials: OpenAI API
   - Model: **gpt-4.1-mini**
   - Add a system instruction similar to:
     - “You are a senior B2B sales strategist… Return ONLY JSON.”
   - Add a user prompt with:
     - Company: `{{$json.Company}}`
     - Website: `{{$json.Website}}`
     - Role: `{{$json.Title}}`
     - Country: `{{$json.Country}}`
   - Ask for JSON with:
     - `category`
     - `lead_score`
     - `priority`
     - `recommended_channel`
     - `reason`
   - Connect: `Loop Over Items → AI-Lead-Analysis`

8. **Create a Code node named `Parse AI`**
   - Type: **Code**
   - Extract text from the OpenAI response object.
   - Parse JSON safely with a try/catch.
   - Map parsed output to:
     - `AI_Category`
     - `AI_LeadScore`
     - `AI_Priority`
     - `Channel`
     - `AI_Reason`
     - `AI_Status`
   - Use defaults if parsing fails.
   - Connect: `AI-Lead-Analysis → Parse AI`

9. **Create a Switch node named `Switch`**
   - Type: **Switch**
   - Add two rules:
     1. `Status` is empty
     2. `Step` equals `1`
   - Recommended implementation:
     - Rule 1 should treat `Status` as a string, not a number.
   - Connect: `Parse AI → Switch`

10. **Create the initial-email OpenAI node named `First AI mail`**
    - Type: **OpenAI**
    - Credentials: same OpenAI credential
    - Model: **gpt-4.1-mini**
    - Prompt should include:
      - `{{$json.Company}}`
      - `{{$json.Person}}`
      - `{{$json.Title}}`
    - Rules:
      - max 2 lines
      - no fluff
      - no AI tone
      - clear benefit
      - avoid “I hope you're doing well”
      - output email body only
    - Connect: `Switch output 0 → First AI mail`

11. **Create a Code node named `Save Message`**
    - Type: **Code**
    - Extract the generated text from the OpenAI response.
    - Save it to `AI_FirstMessage`.
    - Add a fallback such as:
      - `Hi ${item.json.Person}, quick question regarding ${item.json.Company}`
    - Connect: `First AI mail → Save Message`

12. **Create a Gmail node named `Send a message`**
    - Type: **Gmail**
    - Credentials: Gmail OAuth2 or supported Gmail credential
    - Operation: send email
    - To: `{{$json.Email}}`
    - Subject: `Quick question`
    - Message/body: `{{$json.AI_FirstMessage}}`
    - Connect: `Save Message → Send a message`

13. **Create a Google Sheets update node named `Update row in sheet`**
    - Type: **Google Sheets**
    - Operation: **Update**
    - Credentials: same Google Sheets credential
    - Use the same document and sheet as the read node.
    - Matching column: `#`
    - Map fields:
      - `# = {{ $('Get row(s) in sheet').item.json['#'] }}`
      - `Step = 1`
      - `Status = Contacted`
      - `Channel = {{$json.Channel}}`
      - `AI_Category = {{$json.AI_Category}}`
      - `AI_Priority = {{$json.AI_Priority}}`
      - `AI_LeadScore = {{$json.AI_LeadScore}}`
      - `LastContacted = {{new Date().toISOString()}}`
      - `NextActionDate = {{new Date(Date.now() + 2*24*60*60*1000).toISOString()}}`
      - `AI_FirstMessage = {{$json.AI_FirstMessage}}`
    - Connect: `Send a message → Update row in sheet`

14. **Loop back to the batch node**
    - Connect: `Update row in sheet → Loop Over Items`
    - This allows processing of the next item batch.

15. **Create the follow-up OpenAI node named `AI-Follow-up`**
    - Type: **OpenAI**
    - Model: **gpt-4.1-mini**
    - Prompt should ask for a short follow-up email with:
      - Company: `{{$json.Company}}`
      - Friendly tone
      - Not pushy
      - Max 2 lines
      - Output only the email
    - Connect: `Switch output 1 → AI-Follow-up`

16. **Create a Code node named `Parse Follow-up`**
    - Type: **Code**
    - Extract text from the OpenAI response.
    - Save it to `AI_FollowupMessage`.
    - Add fallback:
      - `Just following up on my previous message 🙂`
    - Connect: `AI-Follow-up → Parse Follow-up`

17. **Create a Gmail node named `Send a message1`**
    - Type: **Gmail**
    - Operation: send email
    - To: `{{$json.Email}}`
    - Subject: `Follow-up: "Quick question"`
    - Body: `{{$json.AI_FollowupMessage}}`
    - Connect: `Parse Follow-up → Send a message1`

18. **Create a Google Sheets update node named `Update row in sheet1`**
    - Type: **Google Sheets**
    - Operation: **Update**
    - Matching column: `#`
    - Map:
      - `# = {{ $('Get row(s) in sheet').item.json['#'] }}`
      - `Step = 2`
      - `Status = Follow-up`
      - `LastContacted = {{new Date().toISOString()}}`
      - `NextActionDate = {{new Date(Date.now() + 3*24*60*60*1000).toISOString()}}`
      - `AI_FollowupMessage = {{$json.AI_FollowupMessage}}`
    - Connect: `Send a message1 → Update row in sheet1`

19. **Create a Wait node named `Wait1`**
    - Type: **Wait**
    - Wait time: **2 days**
    - Connect: `Update row in sheet1 → Wait1`

20. **Create an If node named `If1`**
    - Type: **If**
    - Condition: `Response` is empty
    - Expression used in the original workflow:
      - `{{ $('Loop Over Items').item.json.Response }}`
    - Better practice would be to re-read the row from Google Sheets before checking response.
    - Connect: `Wait1 → If1`

21. **Create a Google Sheets update node named `Update row in sheet2`**
    - Type: **Google Sheets**
    - Operation: **Update**
    - Use the same spreadsheet and sheet as the other Google Sheets nodes.
    - Matching column: `#`
    - Map:
      - `# = {{ $('Get row(s) in sheet').item.json['#'] }}`
      - `Step = 3`
      - `Status = Closed`
      - `Response = No replies from the lead till now. So the lead has been closed.`
    - Connect: `If1 (true) → Update row in sheet2`

22. **Loop closed leads back into the batch processor**
    - Connect: `Update row in sheet2 → Loop Over Items`

23. **Add optional sticky notes for documentation**
    - Add one general note describing the workflow purpose and setup.
    - Add section notes for:
      - Step 1 – Trigger and lead intake
      - Step 2 – Data preparation and batching
      - Step 3 – AI lead scoring and routing
      - Step 4 – First outreach automation
      - Step 5 – Follow-up and pipeline completion

24. **Prepare required Google Sheets columns**
    The workflow expects these columns to exist, or very similar names:
    - `#`
    - `Provider Name`
    - `Website`
    - `Country`
    - `First Name`
    - `Title`
    - `Personal Email`
    - `Work Email`
    - `Personal Number ` or `Personal Number`
    - `Work Number`
    - `Status`
    - `Step`
    - `LastContacted`
    - `NextActionDate`
    - `Channel`
    - `AI_Category`
    - `AI_LeadScore`
    - `AI_Priority`
    - `AI_FirstMessage`
    - `AI_FollowupMessage`
    - `Response`

25. **Configure credentials**
    - **Google Sheets**
      - Use OAuth2 or service-account-compatible access as supported by your n8n setup.
      - Ensure the spreadsheet is shared with the credential identity if needed.
    - **OpenAI**
      - Add an API credential with access to `gpt-4.1-mini`.
    - **Gmail**
      - Use Gmail OAuth2 credentials with permission to send mail.

26. **Recommended corrections before production use**
    - Fix `Update row in sheet2` so its spreadsheet and sheet are fully configured.
    - Replace the phone field lookup mismatch:
      - align `Personal Number` vs `Personal Number `
    - Improve the trigger filter to compare `NextActionDate <= now`.
    - Add an email existence check before Gmail nodes.
    - Consider re-reading the Google Sheet row after `Wait1` before closing a lead.
    - Consider adding an explicit default path in the `Switch` node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI lead nurturing and cold email automation with Google Sheets and OpenAI | Workflow branding / purpose |
| Automate lead qualification, personalized outreach, and follow-ups using AI and a Google Sheets CRM. | General overview |
| Reads leads from Google Sheets; uses AI to score and prioritize leads; sends personalized cold emails; generates follow-up messages automatically; tracks outreach status and next steps; updates CRM in real-time. | Functional summary |
| Target users: B2B sales teams, agencies and freelancers, founders scaling outbound outreach. | Intended audience |
| Setup guidance: Connect Google Sheets, OpenAI, Gmail. Ensure columns: Email, Name, Company, Status, Step. Do not hardcode credentials. | Setup note |
| Tip: Use Step + NextActionDate to control multi-step outreach sequences. | Operational design note |

### Additional implementation observations
- There are **no sub-workflows** in this workflow.
- There is **one entry point**: `Schedule Trigger`.
- The workflow uses AI both for **lead qualification** and **message generation**.
- The closure logic is based on the sheet’s `Response` field, not on actual inbox reply detection.
- `Channel` is generated by AI, but only email delivery is implemented; no phone branch exists.