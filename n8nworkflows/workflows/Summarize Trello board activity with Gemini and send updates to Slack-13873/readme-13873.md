Summarize Trello board activity with Gemini and send updates to Slack

https://n8nworkflows.xyz/workflows/summarize-trello-board-activity-with-gemini-and-send-updates-to-slack-13873


# Summarize Trello board activity with Gemini and send updates to Slack

## 1. Workflow Overview

This workflow is designed to generate a daily standup-style summary of Trello board activity using Google Gemini, then distribute that summary through Slack and email, log basic metrics to Google Sheets, and optionally send an overdue-items alert.

Its intended use cases include:

- Daily project reporting for teams using Trello
- Automated standup updates for managers or team leads
- Lightweight board-health monitoring, especially for overdue work
- Tracking reporting metrics over time in Google Sheets

The workflow is logically divided into the following blocks.

### 1.1 Trigger and Runtime Configuration

The workflow starts on a schedule and prepares shared configuration values such as board ID, team email addresses, and Google Sheets document ID.

### 1.2 Trello Data Collection

Two Trello queries run in parallel: one for cards updated today and one for cards created today. Their results are combined into a single structured activity payload.

### 1.3 AI Summary Generation

The formatted Trello activity is sent to Gemini through an HTTP Request node. The AI-generated content is then normalized for downstream delivery.

### 1.4 Distribution and Metric Logging

The generated summary is posted to Slack, emailed to team leads, and logged to Google Sheets.

### 1.5 Overdue Alerting

After logging, the workflow checks whether overdue cards exist. If so, it sends a dedicated Slack alert.

---

## 2. Block-by-Block Analysis

## 2.1 Trigger and Runtime Configuration

**Overview:**  
This block initiates the workflow on a schedule and provides shared settings required by the rest of the workflow. It is the control point for runtime variables such as board ID, email recipients, and sheet destination.

**Nodes Involved:**  
- Daily Standup Trigger
- Configuration Settings

### Node: Daily Standup Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a schedule.
- **Configuration choices:**  
  The node uses a cron-based schedule rule. The JSON only shows a cron-expression rule shell, not the actual expression value, so the exact execution time is not present in the export.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Input: none
  - Output: Configuration Settings
- **Version-specific requirements:**  
  Type version `1.2`
- **Edge cases or potential failure types:**  
  - If no valid cron expression is configured, the trigger will not fire as expected.
  - Timezone mismatches can cause the workflow to run at the wrong local time.
  - Weekday-only behavior described in the sticky note is not guaranteed unless the cron expression explicitly enforces it.
- **Sub-workflow reference:**  
  None

### Node: Configuration Settings

- **Type and technical role:** `n8n-nodes-base.set`  
  Central configuration holder for values consumed by later nodes.
- **Configuration choices:**  
  The exported JSON shows an empty `options` object and does not expose the fields being set. However, downstream expressions indicate this node is expected to provide at least:
  - `teamEmails`
  - `sheetsId`
  - likely `boardId`
  - likely `todayDate`
- **Key expressions or variables used:**  
  Referenced by downstream nodes using:
  - `$('Configuration Settings').item(0).json.teamEmails`
  - `$('Configuration Settings').item(0).json.sheetsId`
- **Input and output connections:**  
  - Input: Daily Standup Trigger
  - Outputs:
    - Get Updated Cards Today
    - Get New Cards Today
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Missing fields will break downstream expressions.
  - Incorrect board ID will cause Trello queries to return wrong or empty data.
  - Invalid email list may cause Gmail send failures.
  - Invalid Sheets document ID will fail append operations.
- **Sub-workflow reference:**  
  None

---

## 2.2 Trello Data Collection

**Overview:**  
This block collects board activity from Trello and reshapes it into a structured payload for AI processing. It also calculates overdue-card statistics.

**Nodes Involved:**  
- Get Updated Cards Today
- Get New Cards Today
- Format Card Activity

### Node: Get Updated Cards Today

- **Type and technical role:** `n8n-nodes-base.trello`  
  Retrieves a set of Trello records, intended here to represent cards updated today.
- **Configuration choices:**  
  The node uses the `getAll` operation. The export does not show additional filters, resource settings, board/list scope, or date constraints, so those details are either omitted or not configured.
- **Key expressions or variables used:**  
  None shown in the export.
- **Input and output connections:**  
  - Input: Configuration Settings
  - Output: Format Card Activity
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Trello credentials may be missing or expired.
  - If no date filter is configured, this may return all cards instead of only cards updated today.
  - Pagination or rate limiting could affect larger boards.
  - Missing expanded list data may cause `card.list?.name` to resolve to `Unknown` later.
- **Sub-workflow reference:**  
  None

### Node: Get New Cards Today

- **Type and technical role:** `n8n-nodes-base.trello`  
  Retrieves a set of Trello records, intended here to represent cards created today.
- **Configuration choices:**  
  Uses the `getAll` operation. As with the updated-cards node, the export does not show the actual filters or constraints needed to isolate “new today” cards.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Input: Configuration Settings
  - Output: Format Card Activity
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Same credential, filter, and pagination issues as the updated-cards node.
  - If “created today” logic is not explicitly configured, data quality will be incorrect.
- **Sub-workflow reference:**  
  None

### Node: Format Card Activity

- **Type and technical role:** `n8n-nodes-base.code`  
  Combines Trello outputs into one normalized JSON object and computes summary metrics including overdue cards.
- **Configuration choices:**  
  The node uses JavaScript with explicit positional access to incoming items:
  - `$input.first()` for configuration
  - `$input.item(1)` for updated cards
  - `$input.item(2)` for new cards
- **Key expressions or variables used:**  
  The code expects:
  - `config.json.todayDate`
  - `config.json.boardId`
  - `updatedCards.json`
  - `newCards.json`

  It creates:
  - `date`
  - `stats`
  - `newCards`
  - `updatedCards`
  - `overdueCards`

  It computes overdue cards by checking:
  - `card.due`
  - `new Date(card.due) < today`

  It also maps selected card fields:
  - `name`
  - `list?.name || 'Unknown'`
  - `shortUrl`
  - `dateLastActivity`
- **Input and output connections:**  
  - Inputs:
    - Configuration Settings
    - Get Updated Cards Today
    - Get New Cards Today
  - Output: Generate AI Summary
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - This node is fragile because it relies on item ordering with `$input.item(1)` and `$input.item(2)`. If the merge behavior is not exactly what the author expected, it may fail or use wrong data.
  - If Trello nodes emit one item per card rather than one array item, `updatedCards.json.length` and `newCards.json.length` will not behave as intended.
  - If `todayDate` or `boardId` is missing in configuration, output will be incomplete.
  - If Trello card objects do not include nested `list` information, list names become `Unknown`.
  - Overdue logic compares with current execution time, not end-of-day semantics.
  - Timezone differences may mark cards overdue too early or too late.
- **Sub-workflow reference:**  
  None

---

## 2.3 AI Summary Generation

**Overview:**  
This block sends the formatted activity data to Gemini and prepares a human-readable summary for distribution. It is the natural-language generation stage of the workflow.

**Nodes Involved:**  
- Generate AI Summary
- Parse AI Response

### Node: Generate AI Summary

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Gemini API directly using HTTP rather than a dedicated AI node.
- **Configuration choices:**  
  - Method is implicitly a body-sending request
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent`
  - Content type: JSON
  - Authentication: generic credential type using HTTP header auth
- **Key expressions or variables used:**  
  The export does not include the request body, headers, or prompt content, but the node is clearly intended to send the formatted Trello activity to Gemini.
- **Input and output connections:**  
  - Input: Format Card Activity
  - Output: Parse AI Response
- **Version-specific requirements:**  
  Type version `4.2`
- **Edge cases or potential failure types:**  
  - Missing or invalid Gemini API key in the header auth credential
  - Incorrect body format for Gemini `generateContent`
  - Model deprecation risk: `gemini-pro` and `v1beta` endpoints may change over time
  - Quota/rate limit errors
  - 400 errors if prompt schema is malformed
  - 401/403 auth errors
  - Empty or unexpected AI response structure
- **Sub-workflow reference:**  
  None

### Node: Parse AI Response

- **Type and technical role:** `n8n-nodes-base.set`  
  Intended to extract the useful summary text and any additional fields needed for Slack, Gmail, Sheets, and overdue checking.
- **Configuration choices:**  
  The export shows no explicit fields, but downstream nodes imply this node should output at least:
  - `summary`
  - `boardId`
  - `date`
  - `hasOverdue`
- **Key expressions or variables used:**  
  Downstream references indicate expectations:
  - Slack likely consumes summary text directly
  - Gmail uses `$json.summary`, `$json.boardId`, `$json.date`
  - IF node uses `$json.hasOverdue`
- **Input and output connections:**  
  - Input: Generate AI Summary
  - Output: Post to Slack
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - If the Gemini response is not parsed into the expected fields, every downstream node may fail.
  - `hasOverdue` must be explicitly set; otherwise the IF condition may evaluate incorrectly.
  - If `boardId` and `date` are not preserved from earlier nodes, the email template will break.
- **Sub-workflow reference:**  
  None

---

## 2.4 Distribution and Metric Logging

**Overview:**  
This block delivers the summary to communication channels and logs data for reporting continuity. It turns the AI output into actionable team communications and historical tracking.

**Nodes Involved:**  
- Post to Slack
- Email Team Leads
- Log to Sheets

### Node: Post to Slack

- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts the generated summary to a Slack channel.
- **Configuration choices:**  
  Operation is `postToChannel`. The export does not reveal the channel ID/name or message expression.
- **Key expressions or variables used:**  
  Not visible in the JSON export, but this node presumably consumes the summary text from Parse AI Response.
- **Input and output connections:**  
  - Input: Parse AI Response
  - Output: Email Team Leads
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Missing Slack credentials or revoked token
  - Invalid or inaccessible target channel
  - Empty message content if AI parsing failed
  - Slack formatting issues if markdown/block payloads are malformed
- **Sub-workflow reference:**  
  None

### Node: Email Team Leads

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the daily summary to a configured list of recipients.
- **Configuration choices:**  
  - `sendTo` is pulled from configuration:
    `{{ $('Configuration Settings').item(0).json.teamEmails }}`
  - Subject:
    `Daily Standup Summary - {{ $json.date }}`
  - Body:
    `Daily Standup Summary\n\n{{ $json.summary }}\n\nView board: https://trello.com/b/{{ $json.boardId }}`
- **Key expressions or variables used:**  
  - `$('Configuration Settings').item(0).json.teamEmails`
  - `$json.summary`
  - `$json.boardId`
  - `$json.date`
- **Input and output connections:**  
  - Input: Post to Slack
  - Output: Log to Sheets
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Gmail OAuth credential errors
  - Invalid recipient string format
  - Missing `summary`, `boardId`, or `date`
  - The Trello URL format assumes `boardId` is valid in board URL context; in Trello, a board short link may work better than an internal ID depending on intended destination
- **Sub-workflow reference:**  
  None

### Node: Log to Sheets

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row to a Google Sheet for tracking summary metrics or history.
- **Configuration choices:**  
  - Operation: `appendRow`
  - Document ID comes from configuration:
    `{{ $('Configuration Settings').item(0).json.sheetsId }}`
  - The specific target sheet/tab and mapped columns are not shown in the export.
- **Key expressions or variables used:**  
  - `$('Configuration Settings').item(0).json.sheetsId`
- **Input and output connections:**  
  - Input: Email Team Leads
  - Output: Check Overdue Items
- **Version-specific requirements:**  
  Type version `4.4`
- **Edge cases or potential failure types:**  
  - Invalid spreadsheet ID
  - Missing permissions on the spreadsheet
  - Column mismatch if append mapping does not match sheet structure
  - If expected fields were not preserved from earlier nodes, the logged row may be incomplete
- **Sub-workflow reference:**  
  None

---

## 2.5 Overdue Alerting

**Overview:**  
This block evaluates whether overdue cards exist and triggers a Slack alert if needed. It acts as a secondary escalation path after the standard summary distribution.

**Nodes Involved:**  
- Check Overdue Items
- Send Overdue Alert

### Node: Check Overdue Items

- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates a boolean flag indicating whether overdue items are present.
- **Configuration choices:**  
  Condition:
  - `value1 = {{ $json.hasOverdue }}`
  - compare to `true`
- **Key expressions or variables used:**  
  - `$json.hasOverdue`
- **Input and output connections:**  
  - Input: Log to Sheets
  - True output: Send Overdue Alert
  - False output: no connected node
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - If `hasOverdue` is undefined, the condition may evaluate false even when overdue cards exist.
  - If `hasOverdue` is a string like `"true"` instead of boolean `true`, matching may fail depending on coercion behavior.
- **Sub-workflow reference:**  
  None

### Node: Send Overdue Alert

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack alert dedicated to overdue Trello cards.
- **Configuration choices:**  
  Operation is `postToChannel`. The export does not show channel or message content.
- **Key expressions or variables used:**  
  Not visible in the export.
- **Input and output connections:**  
  - Input: Check Overdue Items (true branch)
  - Output: none
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Same Slack credential/channel risks as the main Slack node
  - If overdue card details are not included in the payload passed into this node, the alert may be too vague
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation note describing workflow purpose, setup, and customization |  |  | ### How it works<br>This workflow automatically generates daily standup summaries from your Trello boards using Gemini AI. Every business day, it:<br><br>1. Fetches cards updated/created today from your specified Trello board<br>2. Formats the activity data (new cards, moves, comments)<br>3. Uses Gemini AI to generate a human-readable standup summary with highlights and blockers<br>4. Posts the summary to Slack and emails team leads<br>5. Logs metrics to Google Sheets for tracking<br>6. Sends alerts for overdue cards<br><br>### Setup steps<br><br>1. Connect your Trello, Slack, Gmail, and Google Sheets accounts<br>2. Get a Gemini API key from Google AI Studio<br>3. Set your board ID, Slack channel, and team email addresses in the configuration nodes<br>4. Adjust the schedule timing in the trigger (default: 5 PM weekdays)<br>5. Test with a manual execution first<br><br>### Customization<br><br>Modify the Gemini prompt to change summary style, adjust the overdue threshold, or add custom Slack formatting. You can also extend this to multiple boards by duplicating the Trello nodes. |
| Trello Section | Sticky Note | Documentation note for Trello collection block |  |  | ## Trello data collection<br>Fetch today's card updates and new cards from your specified board |
| AI Processing Section | Sticky Note | Documentation note for AI block |  |  | ## AI analysis<br>Gemini processes the data and generates human-readable summary |
| Distribution Section | Sticky Note | Documentation note for distribution/logging block |  |  | ## Distribution & logging<br>Send summary to Slack, email leads, and log metrics to Sheets |
| Alert Section | Sticky Note | Documentation note for overdue alert block |  |  | ## Overdue alerts<br>Check for overdue cards and send alerts if found |
| Daily Standup Trigger | Schedule Trigger | Scheduled workflow entry point |  | Configuration Settings | ### How it works<br>This workflow automatically generates daily standup summaries from your Trello boards using Gemini AI. Every business day, it:<br><br>1. Fetches cards updated/created today from your specified Trello board<br>2. Formats the activity data (new cards, moves, comments)<br>3. Uses Gemini AI to generate a human-readable standup summary with highlights and blockers<br>4. Posts the summary to Slack and emails team leads<br>5. Logs metrics to Google Sheets for tracking<br>6. Sends alerts for overdue cards<br><br>### Setup steps<br><br>1. Connect your Trello, Slack, Gmail, and Google Sheets accounts<br>2. Get a Gemini API key from Google AI Studio<br>3. Set your board ID, Slack channel, and team email addresses in the configuration nodes<br>4. Adjust the schedule timing in the trigger (default: 5 PM weekdays)<br>5. Test with a manual execution first<br><br>### Customization<br><br>Modify the Gemini prompt to change summary style, adjust the overdue threshold, or add custom Slack formatting. You can also extend this to multiple boards by duplicating the Trello nodes. |
| Configuration Settings | Set | Stores runtime configuration values | Daily Standup Trigger | Get Updated Cards Today; Get New Cards Today | ### How it works<br>This workflow automatically generates daily standup summaries from your Trello boards using Gemini AI. Every business day, it:<br><br>1. Fetches cards updated/created today from your specified Trello board<br>2. Formats the activity data (new cards, moves, comments)<br>3. Uses Gemini AI to generate a human-readable standup summary with highlights and blockers<br>4. Posts the summary to Slack and emails team leads<br>5. Logs metrics to Google Sheets for tracking<br>6. Sends alerts for overdue cards<br><br>### Setup steps<br><br>1. Connect your Trello, Slack, Gmail, and Google Sheets accounts<br>2. Get a Gemini API key from Google AI Studio<br>3. Set your board ID, Slack channel, and team email addresses in the configuration nodes<br>4. Adjust the schedule timing in the trigger (default: 5 PM weekdays)<br>5. Test with a manual execution first<br><br>### Customization<br><br>Modify the Gemini prompt to change summary style, adjust the overdue threshold, or add custom Slack formatting. You can also extend this to multiple boards by duplicating the Trello nodes. |
| Get Updated Cards Today | Trello | Retrieves cards intended to represent today’s updated cards | Configuration Settings | Format Card Activity | ## Trello data collection<br>Fetch today's card updates and new cards from your specified board |
| Get New Cards Today | Trello | Retrieves cards intended to represent today’s newly created cards | Configuration Settings | Format Card Activity | ## Trello data collection<br>Fetch today's card updates and new cards from your specified board |
| Format Card Activity | Code | Combines Trello data, computes counts, and prepares AI payload | Get Updated Cards Today; Get New Cards Today | Generate AI Summary | ## Trello data collection<br>Fetch today's card updates and new cards from your specified board |
| Generate AI Summary | HTTP Request | Calls Gemini API to generate standup summary | Format Card Activity | Parse AI Response | ## AI analysis<br>Gemini processes the data and generates human-readable summary |
| Parse AI Response | Set | Extracts normalized summary fields from Gemini response | Generate AI Summary | Post to Slack | ## AI analysis<br>Gemini processes the data and generates human-readable summary |
| Post to Slack | Slack | Sends summary to Slack channel | Parse AI Response | Email Team Leads | ## Distribution & logging<br>Send summary to Slack, email leads, and log metrics to Sheets |
| Email Team Leads | Gmail | Emails the daily summary to recipients | Post to Slack | Log to Sheets | ## Distribution & logging<br>Send summary to Slack, email leads, and log metrics to Sheets |
| Log to Sheets | Google Sheets | Appends summary data to a spreadsheet | Email Team Leads | Check Overdue Items | ## Distribution & logging<br>Send summary to Slack, email leads, and log metrics to Sheets |
| Check Overdue Items | If | Tests whether overdue items exist | Log to Sheets | Send Overdue Alert | ## Overdue alerts<br>Check for overdue cards and send alerts if found |
| Send Overdue Alert | Slack | Sends Slack alert for overdue cards | Check Overdue Items |  | ## Overdue alerts<br>Check for overdue cards and send alerts if found |

---

## 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence based on the exported workflow and the behavior implied by downstream expressions.

1. **Create a new workflow**
   - Title it: **Summarize Trello board activity with Gemini and send updates to Slack**

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: **Daily Standup Trigger**
   - Configure a cron schedule.
   - If you want the behavior described in the notes, set it for business days at 5 PM.
   - Also verify the workflow timezone in n8n settings.

3. **Add a Set node for runtime configuration**
   - Node type: **Set**
   - Name: **Configuration Settings**
   - Add fields such as:
     - `boardId` → your Trello board identifier
     - `todayDate` → expression for current date, for example in ISO or local format
     - `teamEmails` → comma-separated list of email recipients
     - `sheetsId` → target Google Sheets document ID
     - optionally `slackChannel`, `overdueSlackChannel`, or other reusable settings
   - Connect:
     - **Daily Standup Trigger → Configuration Settings**

4. **Add the first Trello node for updated cards**
   - Node type: **Trello**
   - Name: **Get Updated Cards Today**
   - Operation: **Get All**
   - Connect Trello credentials.
   - Configure it to fetch cards from the configured board.
   - Add filters or parameters so it only returns cards updated today.
   - If Trello node filtering is limited, you may need to fetch candidate cards then filter them in code.
   - Connect:
     - **Configuration Settings → Get Updated Cards Today**

5. **Add the second Trello node for new cards**
   - Node type: **Trello**
   - Name: **Get New Cards Today**
   - Operation: **Get All**
   - Use the same Trello credentials.
   - Configure it to fetch cards from the same board, filtered to cards created today.
   - Connect:
     - **Configuration Settings → Get New Cards Today**

6. **Add a Code node to combine and format the activity**
   - Node type: **Code**
   - Name: **Format Card Activity**
   - Connect:
     - **Get Updated Cards Today → Format Card Activity**
     - **Get New Cards Today → Format Card Activity**
   - Add code that:
     - reads config values
     - combines updated and new card results
     - counts total updated, total new, and overdue
     - prepares a compact payload for Gemini
   - The exported code assumes:
     - config is available in one input item
     - updated cards and new cards are available as separate items
   - In practice, you may need to rework this part because n8n does not automatically merge multi-branch inputs into the exact structure expected by `$input.item(1)` and `$input.item(2)`.
   - A safer rebuild is:
     - use explicit **Merge** nodes before the Code node, or
     - read data from named nodes with expressions like `$items('Get Updated Cards Today')`
   - The formatted output should include:
     - `date`
     - `boardId`
     - `stats.totalUpdated`
     - `stats.totalNew`
     - `stats.overdue`
     - `newCards[]`
     - `updatedCards[]`
     - `overdueCards[]`

7. **Add an HTTP Request node for Gemini**
   - Node type: **HTTP Request**
   - Name: **Generate AI Summary**
   - Connect:
     - **Format Card Activity → Generate AI Summary**
   - Configure:
     - Method: `POST`
     - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent`
     - Authentication: **Generic Credential Type**
     - Auth type: **HTTP Header Auth**
   - Create an HTTP Header Auth credential with something like:
     - Header name appropriate for the Gemini API method you use
     - API key from Google AI Studio
   - Set content type to JSON.
   - Build a request body that includes:
     - the structured activity data from the Code node
     - a prompt asking Gemini to produce a concise standup summary, key highlights, blockers, and overdue concerns
   - Because the export omits the body, you must define it manually.
   - Recommended output requirements for the prompt:
     - one human-readable summary paragraph
     - bullet highlights
     - blockers or risks
     - a note if overdue cards exist

8. **Add a Set node to parse the Gemini response**
   - Node type: **Set**
   - Name: **Parse AI Response**
   - Connect:
     - **Generate AI Summary → Parse AI Response**
   - Extract and create fields such as:
     - `summary` → text generated by Gemini
     - `date` → carry forward from earlier data
     - `boardId` → carry forward from earlier data
     - `hasOverdue` → boolean based on overdue count
     - optionally `overdueCards`, `stats`, `slackText`
   - Since the Gemini response structure can vary, inspect one test execution and map the actual response path.
   - If needed, use expressions targeting `candidates[0]...` style fields depending on the API response.

9. **Add a Slack node for the main summary**
   - Node type: **Slack**
   - Name: **Post to Slack**
   - Operation: **Post to Channel**
   - Connect Slack credentials.
   - Configure the target channel.
   - Use the summary field from **Parse AI Response** as the message body.
   - Connect:
     - **Parse AI Response → Post to Slack**

10. **Add a Gmail node for team email**
    - Node type: **Gmail**
    - Name: **Email Team Leads**
    - Connect Gmail credentials.
    - Configure:
      - To: `{{ $('Configuration Settings').item(0).json.teamEmails }}`
      - Subject: `Daily Standup Summary - {{ $json.date }}`
      - Message body:  
        `Daily Standup Summary`  
        blank line  
        `{{ $json.summary }}`  
        blank line  
        `View board: https://trello.com/b/{{ $json.boardId }}`
    - Connect:
      - **Post to Slack → Email Team Leads**
    - Validate that `teamEmails` is a Gmail-compatible recipient string.

11. **Add a Google Sheets node for logging**
    - Node type: **Google Sheets**
    - Name: **Log to Sheets**
    - Operation: **Append Row**
    - Connect Google Sheets credentials.
    - Set document ID with:
      - `{{ $('Configuration Settings').item(0).json.sheetsId }}`
    - Select the sheet/tab to append to.
    - Map columns such as:
      - Date
      - Board ID
      - Summary
      - Total Updated
      - Total New
      - Overdue Count
    - Connect:
      - **Email Team Leads → Log to Sheets**

12. **Add an IF node for overdue detection**
    - Node type: **If**
    - Name: **Check Overdue Items**
    - Connect:
      - **Log to Sheets → Check Overdue Items**
    - Configure a boolean condition:
      - `{{ $json.hasOverdue }}` equals `true`
    - Ensure `hasOverdue` is a real boolean, not a string.

13. **Add a second Slack node for overdue alerts**
    - Node type: **Slack**
    - Name: **Send Overdue Alert**
    - Operation: **Post to Channel**
    - Connect Slack credentials.
    - Configure the target alert channel.
    - Build an alert message that includes:
      - overdue count
      - card names
      - due dates
      - links
    - Connect:
      - **Check Overdue Items (true) → Send Overdue Alert**

14. **Add sticky notes if you want the same visual layout**
    - Create the following notes:
      - **Workflow Overview**
      - **Trello Section**
      - **AI Processing Section**
      - **Distribution Section**
      - **Alert Section**
    - Paste the note content from the exported workflow if you want visual parity.

15. **Test branch by branch**
    - Run manually first.
    - Verify:
      - Trello queries actually return the intended cards for “today”
      - Gemini response path matches your parser
      - Slack formatting is readable
      - Gmail recipients resolve correctly
      - Google Sheets row mapping appends correctly
      - overdue boolean becomes true when expected

16. **Harden the workflow before production**
    - Recommended improvements:
      - add **Merge** nodes before the Code node
      - add error handling branches or an Error Trigger workflow
      - add retries for Gemini, Slack, and Google Sheets
      - validate empty Trello result sets
      - ensure the Trello board URL uses the correct board shortlink if needed

### Credentials required

- **Trello credentials**
  - For both Trello nodes
- **Slack credentials**
  - For both Slack nodes
- **Gmail credentials**
  - For Email Team Leads
- **Google Sheets credentials**
  - For Log to Sheets
- **HTTP Header Auth credential**
  - For Gemini API access in the HTTP Request node

### Important implementation caveat

The exported workflow is conceptually valid, but some required configuration is not visible in the JSON:

- actual cron expression
- actual Set fields in configuration
- Trello filters for “updated today” and “new today”
- Gemini request body and prompt
- Set mappings in Parse AI Response
- Slack message/channel settings
- Google Sheets sheet/tab and column mapping

Also, the current multi-input design into **Format Card Activity** is likely incomplete unless additional merge behavior was configured elsewhere. Rebuilding this part with explicit Merge nodes is strongly recommended.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically generates daily standup summaries from your Trello boards using Gemini AI. Every business day, it fetches cards updated/created today, formats the activity data, generates a human-readable summary with highlights and blockers, posts the summary to Slack, emails team leads, logs metrics to Google Sheets, and sends alerts for overdue cards. | From the workflow overview sticky note |
| Setup guidance: connect Trello, Slack, Gmail, and Google Sheets accounts; get a Gemini API key from Google AI Studio; set your board ID, Slack channel, and team email addresses in configuration nodes; adjust the schedule timing; test manually first. | From the workflow overview sticky note |
| Customization guidance: modify the Gemini prompt to change summary style, adjust the overdue threshold, add custom Slack formatting, or extend to multiple boards by duplicating the Trello nodes. | From the workflow overview sticky note |
| Trello data collection: fetch today's card updates and new cards from your specified board. | Trello section note |
| AI analysis: Gemini processes the data and generates human-readable summary. | AI section note |
| Distribution & logging: send summary to Slack, email leads, and log metrics to Sheets. | Distribution section note |
| Overdue alerts: check for overdue cards and send alerts if found. | Alert section note |