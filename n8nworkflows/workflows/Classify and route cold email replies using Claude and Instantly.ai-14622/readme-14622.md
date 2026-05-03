Classify and route cold email replies using Claude and Instantly.ai

https://n8nworkflows.xyz/workflows/classify-and-route-cold-email-replies-using-claude-and-instantly-ai-14622


# Classify and route cold email replies using Claude and Instantly.ai

# 1. Workflow Overview

This workflow automates the handling of cold email replies received from Instantly.ai. It accepts webhook payloads for incoming replies, extracts the relevant lead and message data, asks Claude to classify the reply intent into one of four categories, then routes the lead into the correct response path.

Primary use cases:
- Automatically identify hot leads who replied positively
- Delay and re-sequence leads who are interested later
- immediately honor unsubscribe requests
- handle referral-style replies with a tailored follow-up
- maintain a tracking log in Google Sheets for all classified outcomes

## 1.1 Input Reception and Normalization

The workflow starts when Instantly.ai sends a `POST` webhook event for a reply. A Set node standardizes incoming fields because webhook payloads may use different property names for the same concept.

## 1.2 AI Intent Classification

The normalized reply body is sent to an Anthropic Claude model via the LangChain LLM Chain node. The prompt forces a one-word output limited to four valid labels:
- `INTERESTED`
- `NOT_NOW`
- `UNSUBSCRIBE`
- `REFERRAL`

## 1.3 Intent Routing

A Switch node reads Claude’s output, normalizes case and whitespace, and routes the item to one of four branches based on exact string matching.

## 1.4 Interested Branch

For positive replies, the workflow optionally posts a Slack alert, sends the lead a calendar-booking email via Gmail, then appends the event to Google Sheets.

## 1.5 Referral Branch

For referral-style replies, the workflow sends a follow-up email requesting an introduction and logs the interaction to Google Sheets.

## 1.6 Not Now Branch

For “later” responses, the workflow logs the reply immediately, waits 30 days, then re-adds the lead to an Instantly sequence through the Instantly API.

## 1.7 Unsubscribe Branch

For opt-out or negative replies, the workflow calls the Instantly unsubscribe API and records the event in Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Normalization

### Overview
This block receives Instantly reply events and maps varying webhook field names into a stable internal structure. It is the foundation for every later branch because all downstream nodes reference these normalized fields.

### Nodes Involved
- Receive Instantly Reply
- Extract Reply Data

### Node Details

#### Receive Instantly Reply
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that exposes an HTTP endpoint Instantly.ai can call when a lead replies.
- **Configuration choices:**
  - Method: `POST`
  - Path: `instantly-reply`
  - Response mode: `lastNode`
- **Key expressions or variables used:** None inside this node.
- **Input and output connections:**
  - Input: none, this is a trigger
  - Output: `Extract Reply Data`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Workflow must be active for production URL generation
  - Incorrect webhook URL in Instantly prevents triggering
  - If Instantly sends an unexpected payload shape, downstream extraction may produce blanks
  - Since response mode is `lastNode`, a long-running branch could delay webhook completion depending on path taken
- **Sub-workflow reference:** None

#### Extract Reply Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes raw webhook data into fixed fields used everywhere else.
- **Configuration choices:**
  - Creates these fields:
    - `reply_body`
    - `lead_email`
    - `lead_first_name`
    - `lead_last_name`
    - `company_name`
    - `campaign_name`
    - `timestamp`
  - Uses fallback chains to support multiple possible webhook field names
- **Key expressions or variables used:**
  - `={{ $json.body.reply_body || $json.body.text || $json.body.message || '' }}`
  - `={{ $json.body.from_email || $json.body.lead_email || $json.body.email || '' }}`
  - `={{ $json.body.first_name || $json.body.lead_first_name || '' }}`
  - `={{ $json.body.last_name || $json.body.lead_last_name || '' }}`
  - `={{ $json.body.company || $json.body.company_name || '' }}`
  - `={{ $json.body.campaign_name || $json.body.campaign || '' }}`
  - `={{ new Date().toISOString() }}`
- **Input and output connections:**
  - Input: `Receive Instantly Reply`
  - Output: `Classify Reply Intent`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Missing `body` object in webhook payload can make expressions resolve unexpectedly
  - Empty strings may propagate into Gmail, Slack, Google Sheets, and API calls
  - Timestamp is generated at workflow runtime, not taken from Instantly event metadata
- **Sub-workflow reference:** None

---

## 2.2 AI Intent Classification

### Overview
This block sends the cleaned reply text to Claude with a strict classification prompt. The model is constrained to return only one valid category, making routing deterministic.

### Nodes Involved
- Classify Reply Intent
- Claude Haiku

### Node Details

#### Classify Reply Intent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  LangChain LLM chain node that builds and sends the prompt to the connected language model.
- **Configuration choices:**
  - Prompt is manually defined
  - The instructions explicitly describe the four categories and require one-word output only
- **Key expressions or variables used:**
  - Main inserted value: `{{ $('Extract Reply Data').item.json.reply_body }}`
- **Input and output connections:**
  - Main input: `Extract Reply Data`
  - AI model input: `Claude Haiku`
  - Main output: `Route by Intent`
- **Version-specific requirements:** Type version `1.4`
  - Requires a compatible n8n installation with LangChain/AI nodes enabled
- **Edge cases or potential failure types:**
  - Anthropic credential/authentication issues
  - Empty reply text can cause weak or ambiguous classification
  - Model output may still deviate from expected labels in rare cases despite the prompt
  - If output includes extra whitespace or case variation, the Switch node compensates using `trim().toUpperCase()`
- **Sub-workflow reference:** None

#### Claude Haiku
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Provides the Anthropic chat model used by the LangChain classification node.
- **Configuration choices:**
  - Model: `claude-haiku-4-5-20251001`
  - Temperature: `0`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via AI language-model connection into `Classify Reply Intent`
- **Version-specific requirements:** Type version `1.3`
  - Requires Anthropic credentials
  - The chosen model name must exist in the n8n version and Anthropic account context being used
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model access not enabled for the account
  - Future model naming changes may require replacement
  - API rate limits or network failures
- **Sub-workflow reference:** None

---

## 2.3 Intent Routing

### Overview
This block evaluates Claude’s result and sends the item to exactly one of four output branches. It depends on the classifier returning a valid category string.

### Nodes Involved
- Route by Intent

### Node Details

#### Route by Intent
- **Type and technical role:** `n8n-nodes-base.switch`  
  Multi-branch router based on exact string comparison.
- **Configuration choices:**
  - Four renamed outputs:
    - `INTERESTED`
    - `NOT_NOW`
    - `UNSUBSCRIBE`
    - `REFERRAL`
  - Each rule compares `{{ $json.text.trim().toUpperCase() }}` to a fixed label
  - Case-insensitive handling is effectively reinforced by explicit normalization
- **Key expressions or variables used:**
  - `={{ $json.text.trim().toUpperCase() }}`
- **Input and output connections:**
  - Input: `Classify Reply Intent`
  - Output 1: `Notify Slack - Hot Lead`
  - Output 2: `Log to Sheets - Not Now`
  - Output 3: `Remove from Instantly Sequences`
  - Output 4: `Send Referral Reply`
- **Version-specific requirements:** Type version `3.2`
- **Edge cases or potential failure types:**
  - If the classifier returns anything outside the four labels, no branch will execute
  - If the LLM output is null or missing, `.trim()` may fail depending on actual runtime output shape
  - No fallback/default branch exists, so unmatched classifications are silently dropped from business processing
- **Sub-workflow reference:** None

---

## 2.4 Interested Branch

### Overview
This branch handles leads who want to know more. It optionally notifies Slack, sends a booking email, and logs the interaction for tracking.

### Nodes Involved
- Notify Slack - Hot Lead
- Send Calendar Link
- Log to Sheets - Interested

### Node Details

#### Notify Slack - Hot Lead
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an alert message to a Slack channel for fast sales follow-up.
- **Configuration choices:**
  - Target selection mode: channel
  - Channel: `#hot-leads`
  - Message includes full lead name, email, and reply text
- **Key expressions or variables used:**
  - `{{ $('Extract Reply Data').item.json.lead_first_name }}`
  - `{{ $('Extract Reply Data').item.json.lead_last_name }}`
  - `{{ $('Extract Reply Data').item.json.lead_email }}`
  - `{{ $('Extract Reply Data').item.json.reply_body }}`
- **Input and output connections:**
  - Input: `Route by Intent` (`INTERESTED` output)
  - Output: `Send Calendar Link`
- **Version-specific requirements:** Type version `2.2`
  - Requires Slack credentials
- **Edge cases or potential failure types:**
  - Missing Slack credentials or channel permissions
  - Channel name may not resolve if workspace naming differs
  - If disabled intentionally, ensure the branch still flows as expected in your n8n version
- **Sub-workflow reference:** None

#### Send Calendar Link
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an outbound email to the interested lead with a scheduling link.
- **Configuration choices:**
  - Recipient from normalized lead email
  - Subject: `Let's find a time to connect`
  - HTML body with greeting, thank-you message, and placeholder calendar link
- **Key expressions or variables used:**
  - `={{ $('Extract Reply Data').item.json.lead_email }}`
  - `{{ $('Extract Reply Data').item.json.lead_first_name }}`
- **Input and output connections:**
  - Input: `Notify Slack - Hot Lead`
  - Output: `Log to Sheets - Interested`
- **Version-specific requirements:** Type version `2.1`
  - Requires Gmail credentials
- **Edge cases or potential failure types:**
  - Invalid or blank email address
  - Gmail quota or OAuth issues
  - Placeholder `YOUR_CALENDAR_LINK` must be replaced
  - HTML formatting may not render exactly the same across email clients
- **Sub-workflow reference:** None

#### Log to Sheets - Interested
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a tracking row for interested leads.
- **Configuration choices:**
  - Operation: `append`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Sheet1`
  - Explicit column mapping
  - Classification hardcoded as `Interested`
- **Key expressions or variables used:**
  - Mapped from `Extract Reply Data`:
    - `Timestamp`
    - `First Name`
    - `Last Name`
    - `Email`
    - `Company`
    - `Reply`
    - `Campaign`
  - Fixed value: `Classification = Interested`
- **Input and output connections:**
  - Input: `Send Calendar Link`
  - Output: none
- **Version-specific requirements:** Type version `4.5`
  - Requires Google Sheets credentials
- **Edge cases or potential failure types:**
  - Wrong document ID
  - Sheet name mismatch
  - Column headers must match the mapping names
  - Google API permission/auth failures
- **Sub-workflow reference:** None

---

## 2.5 Referral Branch

### Overview
This branch addresses replies that redirect you to another person or imply a referral. It sends a polite follow-up email and logs the event.

### Nodes Involved
- Send Referral Reply
- Log to Sheets - Referral

### Node Details

#### Send Referral Reply
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a referral-oriented response to the replying lead.
- **Configuration choices:**
  - Recipient from normalized lead email
  - Subject: `Quick question for you`
  - HTML email asks whether someone in their network might be a better fit
- **Key expressions or variables used:**
  - `={{ $('Extract Reply Data').item.json.lead_email }}`
  - `{{ $('Extract Reply Data').item.json.lead_first_name }}`
- **Input and output connections:**
  - Input: `Route by Intent` (`REFERRAL` output)
  - Output: `Log to Sheets - Referral`
- **Version-specific requirements:** Type version `2.1`
  - Requires Gmail credentials
- **Edge cases or potential failure types:**
  - Blank recipient email
  - Messaging may be inappropriate if the classifier labels a reply as referral incorrectly
  - OAuth or sending-limit failures
- **Sub-workflow reference:** None

#### Log to Sheets - Referral
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends referral interactions to the tracking sheet.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Sheet1`
  - Explicit field mapping
  - Classification hardcoded as `Referral`
- **Key expressions or variables used:**
  - Same extracted lead fields as the other Google Sheets nodes
- **Input and output connections:**
  - Input: `Send Referral Reply`
  - Output: none
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**
  - Same Google Sheets risks as other sheet nodes
  - Classification capitalization differs from the switch labels by design, which is acceptable
- **Sub-workflow reference:** None

---

## 2.6 Not Now Branch

### Overview
This branch stores leads who are not ready yet, then re-enters them into a follow-up sequence after a 30-day delay. It is the only delayed-execution branch in the workflow.

### Nodes Involved
- Log to Sheets - Not Now
- Wait 30 Days
- Re-add to Instantly Sequence

### Node Details

#### Log to Sheets - Not Now
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Logs “not now” responses before any delayed action occurs.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Sheet1`
  - Classification hardcoded as `Not Now`
- **Key expressions or variables used:**
  - Standard field mappings from `Extract Reply Data`
- **Input and output connections:**
  - Input: `Route by Intent` (`NOT_NOW` output)
  - Output: `Wait 30 Days`
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**
  - If this append fails, the delayed resequencing never starts
  - Sheet schema mismatch can halt the branch
- **Sub-workflow reference:** None

#### Wait 30 Days
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses the branch for 30 days before continuing.
- **Configuration choices:**
  - Unit: `days`
  - Amount: `30`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Log to Sheets - Not Now`
  - Output: `Re-add to Instantly Sequence`
- **Version-specific requirements:** Type version `1.1`
  - Requires n8n persistence/execution storage to resume reliably after long waits
- **Edge cases or potential failure types:**
  - If n8n execution data retention is misconfigured, long waits may not resume
  - Workflow deactivation or infrastructure resets can affect delayed continuation depending on deployment setup
- **Sub-workflow reference:** None

#### Re-add to Instantly Sequence
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Instantly API to re-add the lead to a specified follow-up sequence.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.instantly.ai/api/v1/lead/add_to_sequence`
  - Sends JSON-style body parameters including:
    - `api_key`
    - `sequence_id`
    - `email`
    - `first_name`
    - `last_name`
  - Generic credential type set to HTTP Header Auth
  - Header includes `Content-Type: application/json`
- **Key expressions or variables used:**
  - `={{ $('Extract Reply Data').item.json.lead_email }}`
  - `={{ $('Extract Reply Data').item.json.lead_first_name }}`
  - `={{ $('Extract Reply Data').item.json.lead_last_name }}`
- **Input and output connections:**
  - Input: `Wait 30 Days`
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**
  - `YOUR_INSTANTLY_API_KEY` and `YOUR_SEQUENCE_ID` must be replaced
  - API endpoint behavior may change if Instantly updates its API
  - Body includes API key directly; that works functionally but is weaker from a credential-management standpoint
  - If the lead already exists in the sequence, API response may indicate duplication or rejection
- **Sub-workflow reference:** None

---

## 2.7 Unsubscribe Branch

### Overview
This branch removes contacts who ask to opt out and records the action. It is important for compliance and suppression handling.

### Nodes Involved
- Remove from Instantly Sequences
- Log to Sheets - Unsubscribed

### Node Details

#### Remove from Instantly Sequences
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Instantly API unsubscribe endpoint for the lead email.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.instantly.ai/api/v1/lead/unsubscribe`
  - Body parameters:
    - `api_key`
    - `email`
  - Generic credential type: HTTP Header Auth
  - Header: `Content-Type: application/json`
- **Key expressions or variables used:**
  - `={{ $('Extract Reply Data').item.json.lead_email }}`
- **Input and output connections:**
  - Input: `Route by Intent` (`UNSUBSCRIBE` output)
  - Output: `Log to Sheets - Unsubscribed`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**
  - Blank or malformed email
  - API auth failure
  - Unsubscribe endpoint may return errors if the lead is unknown
  - As with the other HTTP node, API key is included in body parameters rather than purely managed via credentials
- **Sub-workflow reference:** None

#### Log to Sheets - Unsubscribed
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Stores a record of unsubscribe events.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Sheet1`
  - Classification hardcoded as `Unsubscribed`
- **Key expressions or variables used:**
  - Standard mapped fields from `Extract Reply Data`
- **Input and output connections:**
  - Input: `Remove from Instantly Sequences`
  - Output: none
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**
  - Logging failure after successful unsubscribe causes incomplete audit trail
  - Spreadsheet schema mismatch
- **Sub-workflow reference:** None

---

## 2.8 Workflow Annotation Nodes

### Overview
These sticky notes do not execute logic, but they document setup, branch purpose, and customization guidance. They are useful operational metadata and should be preserved when recreating the workflow.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for overall workflow purpose, setup checklist, and customization guidance.
- **Configuration choices:** Large introductory note with setup instructions for Instantly, Claude, Google Sheets, Gmail, Slack, and Instantly re-sequencing.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None; informational only
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the receive/extract block
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Claude classification behavior
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Switch routing
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents interested branch and notes Slack is optional
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents referral branch
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents not-now branch and mentions changing the wait duration
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents unsubscribe branch
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Overall workflow documentation and setup checklist |  |  | ## AI Cold Email Reply Classifier & Auto-Router using Claude + Instantly.ai<br>Never manually check your inbox for cold email replies again. This workflow reads every reply, understands the intent using Claude AI, and automatically takes the right action.<br>### How it works<br>1. When a lead replies to your cold email in Instantly.ai, it triggers this workflow instantly via webhook.<br>2. The reply is sent to Claude AI (Haiku), which reads it and labels it as one of four intents: Interested, Not Now, Unsubscribe, or Referral.<br>3. Interested replies get a Slack alert and an automated email with your calendar booking link.<br>4. Not Now replies are logged and automatically followed up after 30 days.<br>5. Unsubscribe replies remove the contact from all Instantly sequences via the API.<br>6. Referral replies get a personalised email asking for an introduction.<br>7. Every reply — regardless of intent — is logged to a Google Sheet for tracking.<br>### Setup steps<br>- [ ] **Instantly.ai webhook** — Go to Instantly → Settings → Integrations → Webhooks → Add Webhook. Paste the URL from the *Receive Instantly Reply* node and set the trigger to *Reply Received*. Activate this workflow first so the URL is generated.<br>- [ ] **Claude AI** — Click the *Claude Haiku* sub-node under *Classify Reply Intent*, create a new credential, and paste your API key from console.anthropic.com. Costs fractions of a cent per reply.<br>- [ ] **Google Sheets** — Create a new sheet with these headers in Row 1: 'Timestamp \| First Name \| Last Name \| Email \| Company \| Reply \| Classification \| Campaign'. Connect your Google account in all four *Log to Sheets* nodes and select your sheet.<br>- [x] **Gmail** — Connect your Gmail account in *Send Calendar Link* and *Send Referral Reply*. Replace 'YOUR_CALENDAR_LINK' with your Calendly or Cal.com link.<br>- [ ] **Slack** *(optional)* — Connect your Slack account in *Notify Slack - Hot Lead* and pick a channel like '#hot-leads'. If you don't use Slack, right-click the node and select Disable.<br>- [ ] **Instantly re-sequence** — In *Re-add to Instantly Sequence*, add your Instantly API key and replace 'YOUR_SEQUENCE_ID' with the ID of your warm follow-up sequence (found in the sequence URL inside Instantly).<br>- [ ] Activate the workflow, send a test reply in Instantly, and confirm all four branches work correctly.<br>### Customization<br>Swap the Slack node for Discord or Telegram if preferred. Change the 30-day wait in the Not Now branch to any delay that suits your sales cycle. |
| Sticky Note1 | stickyNote | Documentation for the receive/extract block |  |  | ## Receive & extract reply<br>Instantly.ai sends the reply here the moment a lead responds. This step pulls out the key details — name, email, company, and reply text — ready for Claude to read. |
| Sticky Note2 | stickyNote | Documentation for AI classification block |  |  | ## Classify intent with Claude<br>Claude Haiku reads the reply and returns one word: INTERESTED, NOT_NOW, UNSUBSCRIBE, or REFERRAL. The model is set to zero temperature so the output is always consistent and predictable. |
| Sticky Note3 | stickyNote | Documentation for routing block |  |  | ## Route by intent<br>Takes Claude's one-word label and sends the reply down the correct branch. Each of the four outputs connects to a different set of actions below. |
| Sticky Note4 | stickyNote | Documentation for interested branch |  |  | ## Interested branch<br>The lead wants to know more or book a call. Sends a Slack alert so you know immediately, emails the lead your calendar link, then logs the reply to Google Sheets.<br>> Slack is optional — right-click the Slack node and select Disable if you don't use it. |
| Sticky Note5 | stickyNote | Documentation for referral branch |  |  | ## Referral branch<br>The lead referred someone else or asked you to contact a different person. Sends them a friendly reply asking for an introduction, then logs the reply to Google Sheets. |
| Sticky Note6 | stickyNote | Documentation for not-now branch |  |  | ## Not now branch<br>The lead isn't ready yet but hasn't said no. Logs the reply immediately, waits 30 days, then automatically re-adds them to a warm follow-up sequence in Instantly. Change the wait duration to suit your sales cycle. |
| Sticky Note7 | stickyNote | Documentation for unsubscribe branch |  |  | ## Unsubscribe branch<br>The lead wants to opt out. Calls the Instantly API to remove them from all active sequences, then logs the event to Google Sheets so you have a record. |
| Receive Instantly Reply | webhook | Receives reply events from Instantly.ai |  | Extract Reply Data | ## Receive & extract reply<br>Instantly.ai sends the reply here the moment a lead responds. This step pulls out the key details — name, email, company, and reply text — ready for Claude to read. |
| Extract Reply Data | set | Normalizes incoming webhook data into stable internal fields | Receive Instantly Reply | Classify Reply Intent | ## Receive & extract reply<br>Instantly.ai sends the reply here the moment a lead responds. This step pulls out the key details — name, email, company, and reply text — ready for Claude to read. |
| Classify Reply Intent | @n8n/n8n-nodes-langchain.chainLlm | Sends reply text to Claude for classification | Extract Reply Data, Claude Haiku | Route by Intent | ## Classify intent with Claude<br>Claude Haiku reads the reply and returns one word: INTERESTED, NOT_NOW, UNSUBSCRIBE, or REFERRAL. The model is set to zero temperature so the output is always consistent and predictable. |
| Claude Haiku | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic chat model backing the classifier |  | Classify Reply Intent | ## Classify intent with Claude<br>Claude Haiku reads the reply and returns one word: INTERESTED, NOT_NOW, UNSUBSCRIBE, or REFERRAL. The model is set to zero temperature so the output is always consistent and predictable. |
| Route by Intent | switch | Routes classified items to one of four intent branches | Classify Reply Intent | Notify Slack - Hot Lead; Log to Sheets - Not Now; Remove from Instantly Sequences; Send Referral Reply | ## Route by intent<br>Takes Claude's one-word label and sends the reply down the correct branch. Each of the four outputs connects to a different set of actions below. |
| Notify Slack - Hot Lead | slack | Alerts Slack about an interested lead | Route by Intent | Send Calendar Link | ## Interested branch<br>The lead wants to know more or book a call. Sends a Slack alert so you know immediately, emails the lead your calendar link, then logs the reply to Google Sheets.<br>> Slack is optional — right-click the Slack node and select Disable if you don't use it. |
| Send Calendar Link | gmail | Sends booking link email to interested lead | Notify Slack - Hot Lead | Log to Sheets - Interested | ## Interested branch<br>The lead wants to know more or book a call. Sends a Slack alert so you know immediately, emails the lead your calendar link, then logs the reply to Google Sheets.<br>> Slack is optional — right-click the Slack node and select Disable if you don't use it. |
| Log to Sheets - Interested | googleSheets | Logs interested replies to Google Sheets | Send Calendar Link |  | ## Interested branch<br>The lead wants to know more or book a call. Sends a Slack alert so you know immediately, emails the lead your calendar link, then logs the reply to Google Sheets.<br>> Slack is optional — right-click the Slack node and select Disable if you don't use it. |
| Send Referral Reply | gmail | Sends referral follow-up email | Route by Intent | Log to Sheets - Referral | ## Referral branch<br>The lead referred someone else or asked you to contact a different person. Sends them a friendly reply asking for an introduction, then logs the reply to Google Sheets. |
| Log to Sheets - Referral | googleSheets | Logs referral replies to Google Sheets | Send Referral Reply |  | ## Referral branch<br>The lead referred someone else or asked you to contact a different person. Sends them a friendly reply asking for an introduction, then logs the reply to Google Sheets. |
| Log to Sheets - Not Now | googleSheets | Logs “not now” replies before delayed follow-up | Route by Intent | Wait 30 Days | ## Not now branch<br>The lead isn't ready yet but hasn't said no. Logs the reply immediately, waits 30 days, then automatically re-adds them to a warm follow-up sequence in Instantly. Change the wait duration to suit your sales cycle. |
| Wait 30 Days | wait | Delays follow-up resequencing by 30 days | Log to Sheets - Not Now | Re-add to Instantly Sequence | ## Not now branch<br>The lead isn't ready yet but hasn't said no. Logs the reply immediately, waits 30 days, then automatically re-adds them to a warm follow-up sequence in Instantly. Change the wait duration to suit your sales cycle. |
| Re-add to Instantly Sequence | httpRequest | Calls Instantly API to add lead into a follow-up sequence | Wait 30 Days |  | ## Not now branch<br>The lead isn't ready yet but hasn't said no. Logs the reply immediately, waits 30 days, then automatically re-adds them to a warm follow-up sequence in Instantly. Change the wait duration to suit your sales cycle. |
| Remove from Instantly Sequences | httpRequest | Calls Instantly API to unsubscribe the lead | Route by Intent | Log to Sheets - Unsubscribed | ## Unsubscribe branch<br>The lead wants to opt out. Calls the Instantly API to remove them from all active sequences, then logs the event to Google Sheets so you have a record. |
| Log to Sheets - Unsubscribed | googleSheets | Logs unsubscribe events to Google Sheets | Remove from Instantly Sequences |  | ## Unsubscribe branch<br>The lead wants to opt out. Calls the Instantly API to remove them from all active sequences, then logs the event to Google Sheets so you have a record. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI Cold Email Reply Classifier & Auto-Router using Claude + Instantly.ai`.

2. **Add a Webhook node**
   - Node type: **Webhook**
   - Name: `Receive Instantly Reply`
   - HTTP Method: `POST`
   - Path: `instantly-reply`
   - Response Mode: `Last Node`
   - Save the node.
   - After activation, copy the production webhook URL into Instantly.ai.

3. **Configure Instantly.ai webhook**
   - In Instantly, go to webhook/integration settings.
   - Add the webhook URL from `Receive Instantly Reply`.
   - Use the event trigger corresponding to reply reception, such as `Reply Received`.

4. **Add a Set node**
   - Node type: **Set**
   - Name: `Extract Reply Data`
   - Connect `Receive Instantly Reply` → `Extract Reply Data`
   - Add these fields:
     1. `reply_body` = `{{ $json.body.reply_body || $json.body.text || $json.body.message || '' }}`
     2. `lead_email` = `{{ $json.body.from_email || $json.body.lead_email || $json.body.email || '' }}`
     3. `lead_first_name` = `{{ $json.body.first_name || $json.body.lead_first_name || '' }}`
     4. `lead_last_name` = `{{ $json.body.last_name || $json.body.lead_last_name || '' }}`
     5. `company_name` = `{{ $json.body.company || $json.body.company_name || '' }}`
     6. `campaign_name` = `{{ $json.body.campaign_name || $json.body.campaign || '' }}`
     7. `timestamp` = `{{ new Date().toISOString() }}`

5. **Add the Anthropic chat model node**
   - Node type: **Anthropic Chat Model** / `lmChatAnthropic`
   - Name: `Claude Haiku`
   - Model: `claude-haiku-4-5-20251001`
   - Temperature: `0`
   - Create/select Anthropic credentials using your API key from `console.anthropic.com`

6. **Add an LLM Chain node**
   - Node type: **Basic LLM Chain** / `chainLlm`
   - Name: `Classify Reply Intent`
   - Connect `Extract Reply Data` → `Classify Reply Intent`
   - Connect `Claude Haiku` to `Classify Reply Intent` as the AI language model
   - Set prompt mode to define text manually
   - Use this prompt logic:
     - Instruct the model that it is a cold email reply classifier
     - Define the four allowed labels:
       - `INTERESTED`
       - `NOT_NOW`
       - `UNSUBSCRIBE`
       - `REFERRAL`
     - Require one word only, no punctuation, no explanation
     - Insert the reply text with `{{ $('Extract Reply Data').item.json.reply_body }}`

7. **Add a Switch node**
   - Node type: **Switch**
   - Name: `Route by Intent`
   - Connect `Classify Reply Intent` → `Route by Intent`
   - Create 4 outputs with renamed output keys:
     1. `INTERESTED`
     2. `NOT_NOW`
     3. `UNSUBSCRIBE`
     4. `REFERRAL`
   - For each rule, compare:
     - Left value: `{{ $json.text.trim().toUpperCase() }}`
     - Operator: string equals
     - Right value: the matching output key

8. **Prepare Google Sheets destination**
   - Create a Google Sheet
   - Add these exact headers in row 1:
     - `Timestamp`
     - `First Name`
     - `Last Name`
     - `Email`
     - `Company`
     - `Reply`
     - `Classification`
     - `Campaign`
   - Copy the spreadsheet ID from the URL
   - Create Google Sheets credentials in n8n

9. **Build the Interested branch: Slack node**
   - Node type: **Slack**
   - Name: `Notify Slack - Hot Lead`
   - Connect `Route by Intent` interested output → `Notify Slack - Hot Lead`
   - Configure:
     - Send message to channel
     - Channel: `#hot-leads` or your preferred sales channel
     - Message body should include:
       - first name
       - last name
       - email
       - reply text
   - Create Slack credentials
   - Optional: disable this node if Slack is not used

10. **Build the Interested branch: Gmail response**
    - Node type: **Gmail**
    - Name: `Send Calendar Link`
    - Connect `Notify Slack - Hot Lead` → `Send Calendar Link`
    - Configure:
      - To: `{{ $('Extract Reply Data').item.json.lead_email }}`
      - Subject: `Let's find a time to connect`
      - HTML body greeting the contact and including your booking link
    - Replace `YOUR_CALENDAR_LINK` with your real Calendly/Cal.com URL
    - Create Gmail OAuth2 credentials

11. **Build the Interested branch: Google Sheets logging**
    - Node type: **Google Sheets**
    - Name: `Log to Sheets - Interested`
    - Connect `Send Calendar Link` → `Log to Sheets - Interested`
    - Operation: `Append`
    - Spreadsheet: your document ID
    - Sheet name: `Sheet1` or your chosen tab
    - Map:
      - `Timestamp` ← extracted timestamp
      - `First Name` ← extracted first name
      - `Last Name` ← extracted last name
      - `Email` ← extracted email
      - `Company` ← extracted company
      - `Reply` ← extracted reply
      - `Campaign` ← extracted campaign
      - `Classification` ← `Interested`

12. **Build the Referral branch: Gmail response**
    - Node type: **Gmail**
    - Name: `Send Referral Reply`
    - Connect `Route by Intent` referral output → `Send Referral Reply`
    - Configure:
      - To: `{{ $('Extract Reply Data').item.json.lead_email }}`
      - Subject: `Quick question for you`
      - HTML body asking whether there is someone in their network who may be a fit

13. **Build the Referral branch: Google Sheets logging**
    - Node type: **Google Sheets**
    - Name: `Log to Sheets - Referral`
    - Connect `Send Referral Reply` → `Log to Sheets - Referral`
    - Operation: `Append`
    - Use the same spreadsheet and column mappings as above
    - Set `Classification` to `Referral`

14. **Build the Not Now branch: Google Sheets logging**
    - Node type: **Google Sheets**
    - Name: `Log to Sheets - Not Now`
    - Connect `Route by Intent` not-now output → `Log to Sheets - Not Now`
    - Operation: `Append`
    - Same spreadsheet mapping
    - Set `Classification` to `Not Now`

15. **Build the Not Now branch: wait**
    - Node type: **Wait**
    - Name: `Wait 30 Days`
    - Connect `Log to Sheets - Not Now` → `Wait 30 Days`
    - Configure:
      - Amount: `30`
      - Unit: `Days`

16. **Build the Not Now branch: Instantly resequence API**
    - Node type: **HTTP Request**
    - Name: `Re-add to Instantly Sequence`
    - Connect `Wait 30 Days` → `Re-add to Instantly Sequence`
    - Configure:
      - Method: `POST`
      - URL: `https://api.instantly.ai/api/v1/lead/add_to_sequence`
      - Send Headers: enabled
      - Send Body: enabled
      - Header:
        - `Content-Type: application/json`
      - Body parameters:
        - `api_key` = your Instantly API key
        - `sequence_id` = your target sequence ID
        - `email` = `{{ $('Extract Reply Data').item.json.lead_email }}`
        - `first_name` = `{{ $('Extract Reply Data').item.json.lead_first_name }}`
        - `last_name` = `{{ $('Extract Reply Data').item.json.lead_last_name }}`
    - If preferred, improve security by moving the API key fully into credentials or environment variables rather than hardcoding it

17. **Build the Unsubscribe branch: Instantly unsubscribe API**
    - Node type: **HTTP Request**
    - Name: `Remove from Instantly Sequences`
    - Connect `Route by Intent` unsubscribe output → `Remove from Instantly Sequences`
    - Configure:
      - Method: `POST`
      - URL: `https://api.instantly.ai/api/v1/lead/unsubscribe`
      - Send Headers: enabled
      - Send Body: enabled
      - Header:
        - `Content-Type: application/json`
      - Body parameters:
        - `api_key` = your Instantly API key
        - `email` = `{{ $('Extract Reply Data').item.json.lead_email }}`

18. **Build the Unsubscribe branch: Google Sheets logging**
    - Node type: **Google Sheets**
    - Name: `Log to Sheets - Unsubscribed`
    - Connect `Remove from Instantly Sequences` → `Log to Sheets - Unsubscribed`
    - Operation: `Append`
    - Same spreadsheet mapping
    - Set `Classification` to `Unsubscribed`

19. **Add optional sticky notes for maintainability**
    - Add notes for:
      - overall setup
      - receive/extract block
      - classify block
      - routing block
      - interested branch
      - referral branch
      - not-now branch
      - unsubscribe branch

20. **Credential setup checklist**
    - **Anthropic:** API key for Claude
    - **Google Sheets:** OAuth2 or service-account access to the destination spreadsheet
    - **Gmail:** OAuth2 with send-mail permission
    - **Slack:** workspace authorization and channel access
    - **Instantly:** API key and valid sequence ID

21. **Test each branch**
    - Send test webhook payloads that simulate:
      - positive reply
      - later/follow-up reply
      - unsubscribe/stop reply
      - referral/contact someone else reply
    - Confirm:
      - classification output is exact
      - switch branch fires properly
      - emails send
      - sheet rows append
      - Instantly API calls succeed

22. **Activate the workflow**
    - Activation is required for the production webhook URL and real-time processing.

### Expected Input
The webhook should send a JSON body containing some or all of the following fields, directly or under `body`:
- `reply_body` or `text` or `message`
- `from_email` or `lead_email` or `email`
- `first_name` or `lead_first_name`
- `last_name` or `lead_last_name`
- `company` or `company_name`
- `campaign_name` or `campaign`

### Expected Outputs
Depending on classification:
- **INTERESTED:** Slack message, booking email, Google Sheets row
- **NOT_NOW:** Google Sheets row, delayed Instantly resequence API call
- **UNSUBSCRIBE:** Instantly unsubscribe API call, Google Sheets row
- **REFERRAL:** referral email, Google Sheets row

### No Sub-workflows
This workflow does not use Execute Workflow nodes or external sub-workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Anthropic API key is obtained from the Anthropic console. | `https://console.anthropic.com` |
| Replace `YOUR_CALENDAR_LINK` with a real scheduling URL such as Calendly or Cal.com. | Your own booking page |
| Replace `YOUR_GOOGLE_SHEET_ID` with the spreadsheet ID from your Google Sheets URL. | Google Sheets document URL |
| Replace `YOUR_INSTANTLY_API_KEY` and `YOUR_SEQUENCE_ID` before production use. | Instantly account and sequence configuration |
| Slack is optional and can be disabled if your team does not use Slack. | Applies to `Notify Slack - Hot Lead` |
| The workflow depends on persistent execution storage for the 30-day wait step to resume reliably. | Applies to `Wait 30 Days` |
| The workflow is currently inactive in the provided JSON. It must be activated for production webhook use. | Workflow-level operational note |