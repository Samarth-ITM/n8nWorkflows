Generate weekly AI business digest from Google Sheets with GPT-4o and send to Slack and email

https://n8nworkflows.xyz/workflows/generate-weekly-ai-business-digest-from-google-sheets-with-gpt-4o-and-send-to-slack-and-email-14332


# Generate weekly AI business digest from Google Sheets with GPT-4o and send to Slack and email

# 1. Workflow Overview

This workflow generates a weekly AI-written business performance digest from Google Sheets data and distributes it to both Slack and email.

Its intended use case is recurring executive or team reporting: every Monday at 8:00 AM, it reads recent business metrics from a Google Sheet, computes week-over-week comparisons, asks OpenAI to convert those metrics into a concise narrative, then sends the result to a Slack channel and by email.

## 1.1 Schedule & Configuration

This block starts the workflow on a weekly schedule and defines the reusable configuration variables needed downstream: Google Sheet ID, sheet name, Slack channel, email recipients, and company name.

## 1.2 Data Retrieval & Metric Computation

This block pulls rows from Google Sheets and calculates aggregated weekly KPIs for the most recent 7 rows versus the previous 7 rows. It also computes derived indicators such as conversion rate and ROAS.

## 1.3 AI Digest Generation

This block sends the computed metrics to OpenAI with a structured prompt so the model can produce a short performance digest with wins, risks, priorities, and outlook.

## 1.4 Formatting & Multi-Channel Delivery

This block converts the AI result into Slack-friendly text and HTML email content, then sends both outputs in parallel.

---

# 2. Block-by-Block Analysis

## 2.1 Schedule & Configuration

**Overview:**  
This block defines when the workflow runs and centralizes the variables that make the workflow reusable without editing every downstream node. It is the entry point for all subsequent processing.

**Nodes Involved:**  
- Trigger every Monday at 8am
- Set report config variables

### Node: Trigger every Monday at 8am

- **Type and technical role:**  
  `n8n-nodes-base.scheduleTrigger`  
  Time-based entry node that launches the workflow automatically.

- **Configuration choices:**  
  Configured to run every week on Monday at 8:00 AM.

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Input: none, this is a trigger node
  - Output: `Set report config variables`

- **Version-specific requirements:**  
  Uses node type version `1.2`. The weekly interval scheduling format must be supported by the installed n8n version.

- **Edge cases or potential failure types:**  
  - Server timezone may differ from business timezone
  - Workflow must be active for schedule-based execution
  - Self-hosted instance downtime at trigger time may prevent execution depending on deployment behavior

- **Sub-workflow reference:**  
  None.

---

### Node: Set report config variables

- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Creates a configuration object used by later nodes.

- **Configuration choices:**  
  Manual assignment mode is used. It includes other fields and adds:
  - `googleSheetId`
  - `sheetName`
  - `slackChannel`
  - `emailRecipients`
  - `companyName`

- **Key expressions or variables used:**  
  Static placeholder values:
  - `YOUR_GOOGLE_SHEET_ID_HERE`
  - `Weekly Metrics`
  - `#business-updates`
  - `user@example.com`
  - `Your Company`

- **Input and output connections:**  
  - Input: `Trigger every Monday at 8am`
  - Output: `Fetch metrics from Google Sheets`

- **Version-specific requirements:**  
  Uses node type version `3.4`. Assignment handling differs across older Set node versions, so the current manual assignment layout should be respected.

- **Edge cases or potential failure types:**  
  - Placeholder values not replaced before activation
  - Invalid Slack channel name
  - Invalid or multiple email addresses not accepted by downstream Gmail settings
  - Wrong Google Sheet ID causing retrieval failure later

- **Sub-workflow reference:**  
  None.

---

## 2.2 Data Retrieval & Metric Computation

**Overview:**  
This block retrieves metrics rows from Google Sheets and converts raw row-level data into a compact summary for the current week and previous week. The logic assumes the latest 14 rows correspond to the last 14 reporting days.

**Nodes Involved:**  
- Fetch metrics from Google Sheets
- Calculate weekly comparisons

### Node: Fetch metrics from Google Sheets

- **Type and technical role:**  
  `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet.

- **Configuration choices:**  
  - Operation: `read`
  - Document ID is taken from `{{$json.googleSheetId}}`
  - Sheet name is taken from `{{$json.sheetName}}`

  No extra filter, range, sort, or limit is configured in the node itself.

- **Key expressions or variables used:**  
  - `={{ $json.sheetName }}`
  - `={{ $json.googleSheetId }}`

- **Input and output connections:**  
  - Input: `Set report config variables`
  - Output: `Calculate weekly comparisons`

- **Version-specific requirements:**  
  Uses node type version `4.5`. Requires a valid Google Sheets OAuth2 credential compatible with this node version.

- **Edge cases or potential failure types:**  
  - OAuth credential expired or missing
  - Sheet ID invalid or inaccessible
  - Sheet name does not exist
  - Headers in the sheet do not match expected field names
  - Empty sheet or fewer than expected rows
  - Read behavior may return all rows unsorted; later logic depends on valid date parsing

- **Sub-workflow reference:**  
  None.

---

### Node: Calculate weekly comparisons

- **Type and technical role:**  
  `n8n-nodes-base.code`  
  JavaScript transformation node that aggregates row data and calculates KPIs.

- **Configuration choices:**  
  The script:
  1. Reads all incoming rows with `$input.all()`
  2. Loads config from `$('Set report config variables').first().json`
  3. Sorts rows descending by `Date` or `date`
  4. Uses first 7 rows as `thisWeek`
  5. Uses next 7 rows as `lastWeek`
  6. Sums key numeric fields
  7. Calculates percentage change
  8. Calculates conversion rate and ROAS
  9. Returns one summarized item

- **Key expressions or variables used:**  
  - `$input.all().map(i => i.json)`
  - `$('Set report config variables').first().json`
  - `new Date(b.Date || b.date)`
  - Fields expected in rows:
    - `Date` or `date`
    - `Revenue`
    - `Leads`
    - `Conversions`
    - `Ad Spend`
    - `Support Tickets`

- **Output structure produced:**  
  Returns:
  - `metrics.thisWeek`
  - `metrics.lastWeek`
  - `metrics.changes`
  - `rawDataPoints`
  - `companyName`
  - `slackChannel`
  - `emailRecipients`
  - `reportDate`

- **Derived metric logic:**  
  - Percentage change:
    - if previous = 0 and current > 0, returns `100`
    - if previous = 0 and current = 0, returns `0`
  - Conversion rate:
    - `conversions / leads * 100`, rounded
  - ROAS:
    - `revenue / adSpend`, rounded to 2 decimals

- **Input and output connections:**  
  - Input: `Fetch metrics from Google Sheets`
  - Output: `Generate AI performance digest`

- **Version-specific requirements:**  
  Uses node type version `2`. Requires JavaScript Code node support in the installed n8n version.

- **Edge cases or potential failure types:**  
  - Fewer than 14 rows leads to incomplete comparisons
  - Invalid date strings may break sorting quality
  - Empty numeric cells become `0` through `parseFloat(...) || 0`
  - Non-standard column names will silently produce incorrect sums
  - If the sheet is not daily and each row is not one day, the “7 rows = 7 days” assumption becomes inaccurate
  - Sorting by parsed date may behave inconsistently if date formatting is locale-specific

- **Sub-workflow reference:**  
  None.

---

## 2.3 AI Digest Generation

**Overview:**  
This block converts the computed metrics into a narrative summary using OpenAI. The model is prompted to produce a concise report with a fixed section structure.

**Nodes Involved:**  
- Generate AI performance digest

### Node: Generate AI performance digest

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.openAi`  
  Chat-based OpenAI node used to generate the business digest text.

- **Configuration choices:**  
  - Resource: `chat`
  - Model: `gpt-4o-mini`
  - Temperature: `0.4`
  - Max tokens: `1000`

  A single prompt message is passed, containing all key metrics and explicit formatting instructions.

- **Key expressions or variables used:**  
  Embedded expressions include:
  - `{{ $json.companyName }}`
  - `{{ $json.metrics.thisWeek.revenue }}`
  - `{{ $json.metrics.changes.revenue }}`
  - `{{ $json.metrics.thisWeek.leads }}`
  - `{{ $json.metrics.changes.leads }}`
  - `{{ $json.metrics.thisWeek.conversions }}`
  - `{{ $json.metrics.changes.conversions }}`
  - `{{ $json.metrics.thisWeek.adSpend }}`
  - `{{ $json.metrics.changes.adSpend }}`
  - `{{ $json.metrics.thisWeek.supportTickets }}`
  - `{{ $json.metrics.changes.supportTickets }}`
  - `{{ $json.metrics.thisWeek.conversionRate }}`
  - `{{ $json.metrics.thisWeek.roas }}`

- **Prompt intent:**  
  The prompt instructs the model to produce:
  1. Headline
  2. Wins
  3. Watch
  4. Priorities
  5. Outlook

  It also requests:
  - under 300 words
  - numerical specificity
  - concise and actionable tone

- **Input and output connections:**  
  - Input: `Calculate weekly comparisons`
  - Output: `Format digest for Slack and email`

- **Version-specific requirements:**  
  Uses node type version `1.8`. Requires the LangChain/OpenAI integration package and a valid OpenAI credential.

- **Edge cases or potential failure types:**  
  - Invalid or missing OpenAI API credentials
  - Model availability changes
  - API quota/rate limit issues
  - Prompt may produce slightly inconsistent formatting
  - If upstream metrics are missing, the model may still generate a narrative using zeros
  - Token limits are unlikely to be reached here, but could if prompt is expanded later

- **Sub-workflow reference:**  
  None.

---

## 2.4 Formatting & Multi-Channel Delivery

**Overview:**  
This block takes the AI output, prepares a Slack text version and an HTML email version, then sends both in parallel. It is the final presentation and distribution stage.

**Nodes Involved:**  
- Format digest for Slack and email
- Post digest to Slack channel
- Email digest to team

### Node: Format digest for Slack and email

- **Type and technical role:**  
  `n8n-nodes-base.code`  
  Builds final channel-specific content from the AI response and upstream metrics.

- **Configuration choices:**  
  The script:
  - Reads AI output from current input item
  - Retrieves prior computed metrics from `Calculate weekly comparisons`
  - Extracts digest text from several possible response fields
  - Builds:
    - `slackMessage`
    - `emailHtml`
    - `emailSubject`
    - plus channel/recipient metadata

- **Key expressions or variables used:**  
  - `$input.first().json`
  - `$('Calculate weekly comparisons').first().json`
  - `aiResponse.message?.content || aiResponse.text || aiResponse.content || 'Report generation failed.'`

- **Formatting logic:**  
  - Slack message begins with `:bar_chart: *Weekly Performance Digest — DATE*`
  - Email HTML includes:
    - heading
    - metric summary box
    - AI digest body rendered in a `white-space: pre-wrap` container

- **Input and output connections:**  
  - Input: `Generate AI performance digest`
  - Outputs:
    - `Post digest to Slack channel`
    - `Email digest to team`

- **Version-specific requirements:**  
  Uses node type version `2`.

- **Edge cases or potential failure types:**  
  - OpenAI response schema may vary by node version; fallback extraction helps but may still fail
  - Digest text is inserted directly into HTML without escaping, so unusual characters or generated markup may affect rendering
  - Slack formatting may not preserve section styling exactly as intended
  - If prior node data cannot be accessed by name, expression lookup will fail

- **Sub-workflow reference:**  
  None.

---

### Node: Post digest to Slack channel

- **Type and technical role:**  
  `n8n-nodes-base.slack`  
  Sends a message to Slack.

- **Configuration choices:**  
  - Resource: `message`
  - Channel from expression: `{{$json.slackChannel}}`
  - Message text from expression: `{{$json.slackMessage}}`

- **Key expressions or variables used:**  
  - `={{ $json.slackMessage }}`
  - `={{ $json.slackChannel }}`

- **Input and output connections:**  
  - Input: `Format digest for Slack and email`
  - Output: none

- **Version-specific requirements:**  
  Uses node type version `2.2`. Requires valid Slack API credentials and permission to post to the target channel.

- **Edge cases or potential failure types:**  
  - Bot not invited to target channel
  - Channel name invalid or private channel inaccessible
  - Slack credential misconfiguration
  - Message length or formatting limitations
  - Rate limiting on frequent use in expanded deployments

- **Sub-workflow reference:**  
  None.

---

### Node: Email digest to team

- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Sends the digest as an HTML email through Gmail.

- **Configuration choices:**  
  - `sendTo`: from `{{$json.emailRecipients}}`
  - `subject`: from `{{$json.emailSubject}}`
  - `message`: from `{{$json.emailHtml}}`
  - `emailType`: `html`

- **Key expressions or variables used:**  
  - `={{ $json.emailRecipients }}`
  - `={{ $json.emailSubject }}`
  - `={{ $json.emailHtml }}`

- **Input and output connections:**  
  - Input: `Format digest for Slack and email`
  - Output: none

- **Version-specific requirements:**  
  Uses node type version `2.1`. Requires Gmail OAuth2 credentials with send permissions.

- **Edge cases or potential failure types:**  
  - OAuth token expired or invalid
  - Gmail sending limits
  - Multiple recipients formatting issues if passed as a plain string in unsupported format
  - HTML may render differently across email clients
  - Organizational Gmail restrictions may block sending

- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger every Monday at 8am | n8n-nodes-base.scheduleTrigger | Weekly workflow trigger |  | Set report config variables | ## 1. Schedule & Config<br>Triggers every Monday at 8am. The config node holds your Sheet ID, Slack channel, and email recipients. |
| Set report config variables | n8n-nodes-base.set | Central configuration storage | Trigger every Monday at 8am | Fetch metrics from Google Sheets | ## 1. Schedule & Config<br>Triggers every Monday at 8am. The config node holds your Sheet ID, Slack channel, and email recipients. |
| Fetch metrics from Google Sheets | n8n-nodes-base.googleSheets | Read source KPI rows from spreadsheet | Set report config variables | Calculate weekly comparisons | ## 2. Data Retrieval & Processing<br>Pulls the last 14 days of metrics from Sheets, then calculates this-week vs. last-week comparisons. |
| Calculate weekly comparisons | n8n-nodes-base.code | Aggregate metrics and compute week-over-week KPIs | Fetch metrics from Google Sheets | Generate AI performance digest | ## 2. Data Retrieval & Processing<br>Pulls the last 14 days of metrics from Sheets, then calculates this-week vs. last-week comparisons. |
| Generate AI performance digest | @n8n/n8n-nodes-langchain.openAi | Generate narrative performance summary with OpenAI | Calculate weekly comparisons | Format digest for Slack and email | ## 3. AI Analysis & Delivery<br>OpenAI generates a narrative digest with trends, wins, concerns, and priorities. Delivered to Slack and email simultaneously. |
| Format digest for Slack and email | n8n-nodes-base.code | Prepare Slack text and HTML email content | Generate AI performance digest | Post digest to Slack channel; Email digest to team | ## 3. AI Analysis & Delivery<br>OpenAI generates a narrative digest with trends, wins, concerns, and priorities. Delivered to Slack and email simultaneously. |
| Post digest to Slack channel | n8n-nodes-base.slack | Send digest to Slack | Format digest for Slack and email |  | ## 3. AI Analysis & Delivery<br>OpenAI generates a narrative digest with trends, wins, concerns, and priorities. Delivered to Slack and email simultaneously. |
| Email digest to team | n8n-nodes-base.gmail | Send digest as HTML email | Format digest for Slack and email |  | ## 3. AI Analysis & Delivery<br>OpenAI generates a narrative digest with trends, wins, concerns, and priorities. Delivered to Slack and email simultaneously. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation / overview |  |  | ## AI Weekly Business Digest → Slack & Email<br><br>Automatically generate a weekly performance report from your Google Sheets data and deliver it to Slack and email.<br><br>### How it works<br>1. **Schedule:** Triggers every Monday at 8am.<br>2. **Fetch:** Pulls the last 14 days of metrics from Google Sheets.<br>3. **Calculate:** Computes this-week vs. last-week comparisons, conversion rate, and ROAS.<br>4. **AI:** OpenAI generates a narrative digest with wins, concerns, and priorities.<br>5. **Deliver:** Posts the formatted report to Slack and sends an HTML email simultaneously.<br><br>### Setup steps<br>1. **Config:** Open the "Set report config variables" node — enter your Sheet ID, sheet name, Slack channel, email recipients, and company name.<br>2. **Data:** Prepare your Google Sheet with columns: Date, Revenue, Leads, Conversions, Ad Spend, Support Tickets.<br>3. **Credentials:** Connect Google Sheets, OpenAI, Slack, and Gmail in each node.<br>4. **Test:** Activate the workflow or trigger it manually to verify. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for schedule/config block |  |  | ## 1. Schedule & Config<br>Triggers every Monday at 8am. The config node holds your Sheet ID, Slack channel, and email recipients. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for data processing block |  |  | ## 2. Data Retrieval & Processing<br>Pulls the last 14 days of metrics from Sheets, then calculates this-week vs. last-week comparisons. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for AI/delivery block |  |  | ## 3. AI Analysis & Delivery<br>OpenAI generates a narrative digest with trends, wins, concerns, and priorities. Delivered to Slack and email simultaneously. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Generate AI weekly business performance digest from Google Sheets with GPT-4o and send to Slack and email`
   - Ensure execution order is standard `v1` if your instance exposes that setting.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name it: `Trigger every Monday at 8am`
   - Configure one weekly rule:
     - Interval field: `weeks`
     - Day: `Monday`
     - Hour: `8`
   - Confirm your workflow/server timezone matches your business reporting expectation.

3. **Add a Set node for configuration**
   - Node type: `Set`
   - Name it: `Set report config variables`
   - Use manual assignment mode.
   - Add these string fields:
     - `googleSheetId` = your Google Sheet ID
     - `sheetName` = `Weekly Metrics` or your tab name
     - `slackChannel` = `#business-updates` or your target channel
     - `emailRecipients` = one or more recipient emails
     - `companyName` = your company display name
   - Keep “Include Other Fields” enabled.
   - Connect:
     - `Trigger every Monday at 8am` → `Set report config variables`

4. **Prepare the Google Sheet**
   - Create a Google Sheet with a tab matching `sheetName`.
   - Add column headers exactly as expected:
     - `Date`
     - `Revenue`
     - `Leads`
     - `Conversions`
     - `Ad Spend`
     - `Support Tickets`
   - Add at least 14 rows of recent daily data.
   - Prefer a consistent date format recognized by JavaScript date parsing, such as `YYYY-MM-DD`.

5. **Add a Google Sheets node**
   - Node type: `Google Sheets`
   - Name it: `Fetch metrics from Google Sheets`
   - Operation: `Read`
   - Set:
     - `Document ID` = expression `{{ $json.googleSheetId }}`
     - `Sheet Name` = expression `{{ $json.sheetName }}`
   - Leave additional options empty unless you need custom behavior.
   - Create or attach Google Sheets OAuth2 credentials with access to the spreadsheet.
   - Connect:
     - `Set report config variables` → `Fetch metrics from Google Sheets`

6. **Add a Code node for KPI calculation**
   - Node type: `Code`
   - Name it: `Calculate weekly comparisons`
   - Language: JavaScript
   - Paste logic equivalent to:
     - collect all incoming rows
     - sort descending by `Date` or `date`
     - split top 7 rows as current week and next 7 rows as previous week
     - sum `Revenue`, `Leads`, `Conversions`, `Ad Spend`, `Support Tickets`
     - calculate percent changes
     - calculate conversion rate and ROAS
     - return one summary item with config values and `reportDate`

   - Recreate these behaviors specifically:
     - If previous value is `0` and current is positive, percentage change returns `100`
     - Conversion rate defaults to `0` when leads are `0`
     - ROAS defaults to `0` when ad spend is `0`

   - Include references to the Set node so the final output contains:
     - `companyName`
     - `slackChannel`
     - `emailRecipients`
     - `reportDate`

   - Connect:
     - `Fetch metrics from Google Sheets` → `Calculate weekly comparisons`

7. **Add an OpenAI Chat node**
   - Node type: `OpenAI` via LangChain integration
   - Name it: `Generate AI performance digest`
   - Resource: `Chat`
   - Model: `gpt-4o-mini`
   - Options:
     - `temperature` = `0.4`
     - `maxTokens` = `1000`
   - Add one message containing a prompt that:
     - identifies the AI as a business analyst
     - injects company name and all computed metrics
     - asks for these sections:
       1. HEADLINE
       2. WINS
       3. WATCH
       4. PRIORITIES
       5. OUTLOOK
     - limits the response to under 300 words
     - asks for specific numbers

   - Use expressions for all metric values from the previous node.
   - Attach valid OpenAI API credentials.
   - Connect:
     - `Calculate weekly comparisons` → `Generate AI performance digest`

8. **Add a Code node for final formatting**
   - Node type: `Code`
   - Name it: `Format digest for Slack and email`
   - Language: JavaScript
   - Recreate logic that:
     - reads the AI response
     - gets the summary object from `Calculate weekly comparisons`
     - extracts text from possible fields such as:
       - `message.content`
       - `text`
       - `content`
     - falls back to `Report generation failed.` if no output exists

   - Build these outputs:
     - `slackMessage`
       - starts with `:bar_chart: *Weekly Performance Digest — YYYY-MM-DD*`
       - then appends the AI digest
     - `emailSubject`
       - `Weekly Performance Digest — YYYY-MM-DD`
     - `emailHtml`
       - include a heading
       - include a “Key Metrics” block
       - include the AI digest body
     - also pass through:
       - `slackChannel`
       - `emailRecipients`
       - `reportDate`

   - Connect:
     - `Generate AI performance digest` → `Format digest for Slack and email`

9. **Add a Slack node**
   - Node type: `Slack`
   - Name it: `Post digest to Slack channel`
   - Resource: `Message`
   - Channel: expression `{{ $json.slackChannel }}`
   - Text: expression `{{ $json.slackMessage }}`
   - Attach Slack credentials for a bot or app that can post to the channel.
   - Make sure the bot is invited to the channel if required.
   - Connect:
     - `Format digest for Slack and email` → `Post digest to Slack channel`

10. **Add a Gmail node**
    - Node type: `Gmail`
    - Name it: `Email digest to team`
    - Operation: send email
    - Set:
      - `Send To` = expression `{{ $json.emailRecipients }}`
      - `Subject` = expression `{{ $json.emailSubject }}`
      - `Message` = expression `{{ $json.emailHtml }}`
      - `Email Type` = `HTML`
    - Attach Gmail OAuth2 credentials with send permission.
    - Connect:
      - `Format digest for Slack and email` → `Email digest to team`

11. **Optional: add sticky notes for maintainability**
    - Add one general overview note describing the full workflow.
    - Add one note above the trigger/config nodes.
    - Add one note above the Google Sheets/calculation nodes.
    - Add one note above the OpenAI/output nodes.

12. **Credential setup checklist**
    - **Google Sheets OAuth2**
      - Must allow read access to the spreadsheet
      - The connected Google account must have access to the file
    - **OpenAI API**
      - Must have model access for `gpt-4o-mini`
      - Ensure quota is available
    - **Slack API**
      - Must permit posting messages
      - Bot/app must have access to target channel
    - **Gmail OAuth2**
      - Must permit sending mail
      - Be aware of sender limits and organization restrictions

13. **Test the workflow manually**
    - Run the workflow from the trigger or from the Set node during setup.
    - Confirm:
      - Google Sheets returns rows
      - calculation node outputs current and previous week metrics
      - OpenAI returns a structured digest
      - Slack receives the message
      - Gmail sends a formatted HTML email

14. **Validate assumptions before production use**
    - Ensure the newest rows really represent the last 14 days
    - Ensure each row corresponds to one day
    - Ensure the sheet contains at least 14 valid entries every Monday
    - Ensure dates sort correctly

15. **Activate the workflow**
    - Once all credentials and test data are validated, activate the workflow so the schedule trigger runs automatically each Monday.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Weekly Business Digest → Slack & Email. Automatically generate a weekly performance report from your Google Sheets data and deliver it to Slack and email. | Overall workflow purpose |
| How it works: 1. Schedule every Monday at 8am. 2. Fetch last 14 days from Google Sheets. 3. Calculate week-over-week metrics, conversion rate, and ROAS. 4. Use OpenAI to generate narrative digest. 5. Deliver to Slack and email. | Overall workflow logic |
| Setup reminder: open the “Set report config variables” node and enter your Sheet ID, sheet name, Slack channel, email recipients, and company name. | Initial configuration |
| Data requirement: prepare Google Sheet columns `Date, Revenue, Leads, Conversions, Ad Spend, Support Tickets`. | Source data contract |
| Credential reminder: connect Google Sheets, OpenAI, Slack, and Gmail credentials in their respective nodes. | Deployment prerequisite |
| Test reminder: activate the workflow or trigger it manually to verify. | Validation step |

**Disclaimer:** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.