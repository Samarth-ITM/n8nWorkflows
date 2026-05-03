Generate IPL post‑match and weekly email analyses with GPT‑4o, CricAPI and Gmail

https://n8nworkflows.xyz/workflows/generate-ipl-post-match-and-weekly-email-analyses-with-gpt-4o--cricapi-and-gmail-14369


# Generate IPL post‑match and weekly email analyses with GPT‑4o, CricAPI and Gmail

# 1. Workflow Overview

This workflow automates two related IPL cricket content pipelines:

1. **Post-match analysis pipeline**: every 30 minutes, it checks CricAPI for newly completed IPL T20 matches, detects matches that have not yet been analyzed, generates a structured journalist-style tactical analysis with GPT-4o-mini, sends the result by Gmail, and records the outcome in Google Sheets.
2. **Weekly digest pipeline**: every Monday at 9:00 AM, it reads the week’s generated analyses from Google Sheets, asks GPT-4o-mini to write a roundup newsletter, formats it as HTML, and emails it.

The workflow is designed for:
- Cricket media automation
- Newsletter generation
- Match recap operations
- AI-assisted sports reporting
- Scheduled fan engagement emails

## 1.1 Match Detection and De-duplication
This block polls CricAPI, reads the existing match log from Google Sheets, and identifies one completed IPL T20 match that has not yet been processed.

## 1.2 Match Registration
Once a new match is found, it is appended to the Match Log sheet with tracking flags indicating analysis has not yet been completed.

## 1.3 Post-Match AI Analysis and Email Delivery
The workflow builds a detailed prompt from match metadata and score information, sends it to OpenAI, parses the structured JSON response, renders a branded HTML email, and sends it through Gmail.

## 1.4 Logging and State Update
After the post-match email is sent, the workflow updates the Match Log to mark the match as analyzed and logs a summary entry into the Analysis Log sheet.

## 1.5 Weekly Digest Generation
Every Monday, the workflow reads the Analysis Log, filters analyses from the last 7 days, builds a weekly summary prompt, asks OpenAI to produce a digest in JSON format, converts it to HTML, and sends the weekly roundup email.

---

# 2. Block-by-Block Analysis

## 2.1 Match Detection and De-duplication

**Overview:**  
This block runs on a 30-minute schedule, fetches current/recent matches from CricAPI, reads the historical match log, and determines whether there is a newly completed IPL T20 match that has not yet been analyzed. If no such match exists, it sets a skip flag so downstream processing can stop cleanly.

**Nodes Involved:**  
- Every 30 Min Trigger
- Fetch Recent Matches
- Read Match Log
- Find New Completed IPL Match
- IF New Match Found

### Node: Every 30 Min Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger` — time-based workflow entry point.
- **Configuration choices:** Configured to run every 30 minutes.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch Recent Matches**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Workflow must be active for schedule execution.
  - Server timezone affects exact run times.
- **Sub-workflow reference:** None.

### Node: Fetch Recent Matches
- **Type and technical role:** `n8n-nodes-base.httpRequest` — fetches current/recent match data from CricAPI.
- **Configuration choices:**
  - GET request to `https://api.cricapi.com/v1/currentMatches`
  - Query parameters:
    - `apikey=your_api_key`
    - `offset=0`
- **Key expressions or variables used:** Static query values.
- **Input and output connections:** Input from **Every 30 Min Trigger**; output to **Read Match Log**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Invalid or missing CricAPI key
  - Rate limits or quota exhaustion
  - API schema changes
  - Empty `data` array
  - Network timeouts
- **Sub-workflow reference:** None.

### Node: Read Match Log
- **Type and technical role:** `n8n-nodes-base.googleSheets` — reads previously tracked matches from Google Sheets.
- **Configuration choices:**
  - Reads from document `IPL Post Match Analyzer`
  - Sheet: `Match Log`
  - `alwaysOutputData: true`, which helps preserve flow behavior even when the sheet is empty
- **Key expressions or variables used:** None directly in configuration.
- **Input and output connections:** Input from **Fetch Recent Matches**; output to **Find New Completed IPL Match**.
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Missing spreadsheet or wrong sheet tab
  - OAuth token expiration
  - Sheet permissions issue
  - Empty sheet on first run
- **Sub-workflow reference:** None.

### Node: Find New Completed IPL Match
- **Type and technical role:** `n8n-nodes-base.code` — compares CricAPI results against the Match Log and selects a new completed match.
- **Configuration choices:**
  - Reads CricAPI response from `$('Fetch Recent Matches').first().json`
  - Reads all match log rows from `$('Read Match Log').all()`
  - Builds a `Set` of already analyzed `match_id` values where `analyzed` is `true` or `'true'`
  - Selects first match that satisfies:
    - name includes `IPL` or `INDIAN PREMIER LEAGUE`
    - `matchType === 't20'`
    - `matchStarted === true`
    - `matchEnded === true`
    - not already analyzed
  - Includes fallback logic for testing: any completed T20 with at least two score entries
  - Returns `{ skip: true, reason: 'No new completed matches' }` if nothing qualifies
  - Extracts innings score summaries and key fields for downstream prompting
- **Key expressions or variables used:**
  - `$('Fetch Recent Matches').first().json`
  - `$('Read Match Log').all()`
  - fields like `match.id`, `match.name`, `match.status`, `match.score`
- **Input and output connections:** Input from **Read Match Log**; outputs to both **IF New Match Found** and **Build Analysis Prompt**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - CricAPI response missing expected fields
  - Match score array incomplete
  - Match names not containing expected IPL wording
  - Fallback may process a non-IPL completed T20 during testing if no IPL match matches criteria
  - Duplicate detection depends on exact `match_id` consistency
- **Sub-workflow reference:** None.

### Node: IF New Match Found
- **Type and technical role:** `n8n-nodes-base.if` — gates the logging branch based on the `skip` flag.
- **Configuration choices:**
  - Condition: `{{$json.skip}} != "true"`
  - Loose type validation enabled
- **Key expressions or variables used:** `={{ $json.skip }}`
- **Input and output connections:** Input from **Find New Completed IPL Match**; true branch outputs to **Save to Match Log**. No false branch is connected.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Because comparison is string-based, non-boolean/string values could behave unexpectedly
  - False path intentionally terminates
- **Sub-workflow reference:** None.

---

## 2.2 Match Registration

**Overview:**  
This block records the newly found match in the Match Log before analysis is sent. It creates a durable tracking row so the match can be marked analyzed later and so repeat detections can be prevented.

**Nodes Involved:**  
- Save to Match Log

### Node: Save to Match Log
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends a new row to the Match Log sheet.
- **Configuration choices:**
  - Operation: `append`
  - Target sheet: `Match Log`
  - Mapped columns:
    - `match_id = {{$json.matchId}}`
    - `match_name = {{$json.matchName}}`
    - `date = {{$json.matchDate}}`
    - `winner = {{$json.status}}`
    - `analyzed = false`
    - `analysis_sent = false`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `={{ $json.matchDate }}`
  - `={{ $json.status }}`
  - `={{ $json.matchId }}`
  - `={{ $json.matchName }}`
- **Input and output connections:** Input from **IF New Match Found**. No downstream connection from this node.
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Duplicate rows can occur if detection logic and sheet state fall out of sync
  - Column names in the sheet must exactly match mapping
  - Auth or permission errors
- **Sub-workflow reference:** None.

---

## 2.3 Post-Match AI Analysis and Email Delivery

**Overview:**  
This block converts score and result data into a structured prompt, sends it to OpenAI for cricket-style analysis, parses the JSON response, transforms it into a polished HTML email, and sends that email through Gmail.

**Nodes Involved:**  
- Build Analysis Prompt
- Match Analyst
- Parse Analysis & Build Email
- Send Analysis Email

### Node: Build Analysis Prompt
- **Type and technical role:** `n8n-nodes-base.code` — constructs the prompt for the model from normalized match fields.
- **Configuration choices:**
  - Reads match data from `$('Find New Completed IPL Match').first().json`
  - Includes:
    - match name
    - result string
    - both team scores
    - runs, wickets, overs
    - computed run rates
  - Requests a strict JSON output with keys:
    - `headline`
    - `match_summary`
    - `innings1_analysis`
    - `innings2_analysis`
    - `key_moments`
    - `tactical_breakdown`
    - `player_of_match`
    - `what_to_watch_next`
    - `analysis_summary`
- **Key expressions or variables used:**
  - `$('Find New Completed IPL Match').first().json`
  - inline run-rate formula:
    - `data.innings1Overs > 0 ? ... : 'N/A'`
    - `data.innings2Overs > 0 ? ... : 'N/A'`
- **Input and output connections:** Input from **Find New Completed IPL Match**; output to **Match Analyst**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `skip: true` also reaches this node, it will still build a prompt from incomplete data because this branch is not gated by the IF node
  - Missing or malformed score fields may produce `0` or `N/A`
  - Team arrays shorter than expected fall back to default labels earlier in the flow
- **Sub-workflow reference:** None.

### Node: Match Analyst
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi` — sends the analysis prompt to OpenAI.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Response input includes only one content item: `{{$json.prompt}}`
  - No additional system message configured in this node
- **Key expressions or variables used:** `={{ $json.prompt }}`
- **Input and output connections:** Input from **Build Analysis Prompt**; output to **Parse Analysis & Build Email**.
- **Version-specific requirements:** Type version `2.1`; requires OpenAI credentials and compatible n8n LangChain/OpenAI node package.
- **Edge cases or potential failure types:**
  - OpenAI auth failure
  - Rate limits
  - Non-JSON or partial JSON responses
  - Model output structure drift
- **Sub-workflow reference:** None.

### Node: Parse Analysis & Build Email
- **Type and technical role:** `n8n-nodes-base.code` — parses AI JSON and generates branded post-match HTML email content.
- **Configuration choices:**
  - Reads response text from `items[0].json.output[0].content[0].text`
  - Removes optional markdown fences via regex
  - Parses JSON with `JSON.parse(cleaned)`
  - Pulls original match context from `$('Build Analysis Prompt').first().json`
  - Builds a styled HTML email featuring:
    - match title and date
    - scorecard panel
    - headline and summary
    - innings breakdowns
    - key moments list
    - tactical breakdown
    - player of the match
    - what to watch next
  - Returns:
    - `headline`
    - `match_summary`
    - `player_of_match`
    - `analysis_summary`
    - `htmlEmail`
    - `subject`
- **Key expressions or variables used:**
  - `items[0].json.output[0].content[0].text`
  - `$('Build Analysis Prompt').first().json`
- **Input and output connections:** Input from **Match Analyst**; output to **Send Analysis Email**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `JSON.parse` failure if model does not return valid JSON
  - Unexpected output schema from OpenAI node
  - Missing `key_moments` array
  - HTML injection risk if upstream content is not trusted
- **Sub-workflow reference:** None.

### Node: Send Analysis Email
- **Type and technical role:** `n8n-nodes-base.gmail` — sends the post-match HTML email.
- **Configuration choices:**
  - Recipient: `your_email_id`
  - Subject: `{{$json.subject}}`
  - Message body: `{{$json.htmlEmail}}`
- **Key expressions or variables used:**
  - `={{ $json.htmlEmail }}`
  - `={{ $json.subject }}`
- **Input and output connections:** Input from **Parse Analysis & Build Email**; output to **Update Match Log**.
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Gmail auth failure
  - Recipient misconfiguration
  - Gmail sending quota or policy restrictions
  - HTML rendering differences in email clients
- **Sub-workflow reference:** None.

---

## 2.4 Logging and State Update

**Overview:**  
After successful email dispatch, this block updates the tracked match row to mark it analyzed and logs the generated analysis summary in a separate sheet for weekly reuse.

**Nodes Involved:**  
- Update Match Log
- Log to Analysis Log

### Node: Update Match Log
- **Type and technical role:** `n8n-nodes-base.googleSheets` — updates an existing Match Log row using `match_id` as the key.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `match_id`
  - Updated values:
    - `match_id = {{$('Parse Analysis & Build Email').item.json.matchId}}`
    - `analyzed = true`
    - `analysis_sent = true`
  - `executeOnce: true`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `={{ $('Parse Analysis & Build Email').item.json.matchId }}`
- **Input and output connections:** Input from **Send Analysis Email**; output to **Log to Analysis Log**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - If the row was never appended earlier, update may fail or affect no rows
  - `match_id` type mismatch can prevent matching
  - Sheet schema drift
- **Sub-workflow reference:** None.

### Node: Log to Analysis Log
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends a compact analysis record for later digest generation.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Analysis Log`
  - Values:
    - `date = current ISO date`
    - `match_name = {{$('Parse Analysis & Build Email').item.json.matchName}}`
    - `winner = {{$('Parse Analysis & Build Email').item.json.status}}`
    - `player_of_match = {{$('Parse Analysis & Build Email').item.json.player_of_match}}`
    - `analysis_summary = {{$('Parse Analysis & Build Email').item.json.analysis_summary}}`
    - `email_sent = true`
  - `executeOnce: true`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString().split('T')[0] }}`
  - `={{ $('Parse Analysis & Build Email').item.json.status }}`
  - `={{ $('Parse Analysis & Build Email').item.json.matchName }}`
  - `={{ $('Parse Analysis & Build Email').item.json.player_of_match }}`
  - `={{ $('Parse Analysis & Build Email').item.json.analysis_summary }}`
- **Input and output connections:** Input from **Update Match Log**. No downstream connection.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Current date is logging execution date, not match date
  - Missing parsed values from AI output can leave blank columns
  - Auth and sheet permission issues
- **Sub-workflow reference:** None.

---

## 2.5 Weekly Digest Generation

**Overview:**  
This branch runs once per week, reads all saved analysis summaries, filters them to the last seven days, generates a roundup prompt, sends it to OpenAI, converts the JSON response into an email, and sends the digest.

**Nodes Involved:**  
- Weekly Monday 9AM Trigger
- Read Analysis Log
- Build Weekly Digest Prompt
- IF Matches This Week
- Weekly Journalist
- Parse & Build Weekly Email
- Send Weekly Digest

### Node: Weekly Monday 9AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger` — weekly entry point.
- **Configuration choices:**
  - Runs every week
  - Day: Monday
  - Hour: 9
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Read Analysis Log**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Server timezone affects actual delivery time
  - Workflow must be active
- **Sub-workflow reference:** None.

### Node: Read Analysis Log
- **Type and technical role:** `n8n-nodes-base.googleSheets` — reads all prior analysis entries.
- **Configuration choices:**
  - Reads from sheet `Analysis Log`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Weekly Monday 9AM Trigger**; output to **Build Weekly Digest Prompt**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Empty sheet
  - Auth failures
  - Date values not parseable by JavaScript
- **Sub-workflow reference:** None.

### Node: Build Weekly Digest Prompt
- **Type and technical role:** `n8n-nodes-base.code` — filters analysis rows to the last 7 days and builds the weekly journalist prompt.
- **Configuration choices:**
  - Reads all incoming items
  - Returns `skip: true` if there are zero rows
  - Computes `sevenDaysAgo`
  - Filters rows where `new Date(a.json.date) >= sevenDaysAgo`
  - Builds multi-match summaries including:
    - match name
    - result
    - player of match
    - analysis summary
  - Requests JSON output with:
    - `week_headline`
    - `week_intro`
    - `match_recaps`
    - `player_of_week`
    - `week_talking_point`
    - `next_week_preview`
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:** Input from **Read Analysis Log**; output to **IF Matches This Week**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Invalid dates can exclude rows unexpectedly
  - Timezone/date boundaries may include or exclude a marginal match
  - If `analysis_summary` is missing, prompt quality drops
- **Sub-workflow reference:** None.

### Node: IF Matches This Week
- **Type and technical role:** `n8n-nodes-base.if` — prevents weekly email generation when the digest block returns `skip: true`.
- **Configuration choices:**
  - Condition: `{{$json.skip}} != "true"`
  - Loose type validation enabled
- **Key expressions or variables used:** `={{ $json.skip }}`
- **Input and output connections:** Input from **Build Weekly Digest Prompt**; true branch outputs to **Weekly Journalist**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Same string/boolean comparison caveat as the post-match IF
- **Sub-workflow reference:** None.

### Node: Weekly Journalist
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi` — generates the weekly digest text in JSON form.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - User content: `{{$json.prompt}}`
  - System instruction: “You are an expert IPL cricket journalist. Always respond in valid JSON only. No markdown, no code blocks.”
- **Key expressions or variables used:** `={{ $json.prompt }}`
- **Input and output connections:** Input from **IF Matches This Week**; output to **Parse & Build Weekly Email**.
- **Version-specific requirements:** Type version `2.1`; requires OpenAI credentials.
- **Edge cases or potential failure types:**
  - Invalid JSON despite system instruction
  - Rate limits/auth failures
  - Output schema mismatch
- **Sub-workflow reference:** None.

### Node: Parse & Build Weekly Email
- **Type and technical role:** `n8n-nodes-base.code` — parses weekly AI output and generates branded HTML digest email.
- **Configuration choices:**
  - Reads response text from `items[0].json.output[0].content[0].text`
  - Removes markdown fences
  - Parses JSON
  - Reads context from `$('Build Weekly Digest Prompt').first().json`
  - Creates HTML sections for:
    - headline
    - intro
    - per-match recap cards
    - player of the week
    - talking point
    - next week preview
  - Builds email subject from `week_headline`
- **Key expressions or variables used:**
  - `items[0].json.output[0].content[0].text`
  - `$('Build Weekly Digest Prompt').first().json`
- **Input and output connections:** Input from **Weekly Journalist**; output to **Send Weekly Digest**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `JSON.parse` errors
  - Missing `match_recaps` array
  - HTML injection if content source is not trusted
- **Sub-workflow reference:** None.

### Node: Send Weekly Digest
- **Type and technical role:** `n8n-nodes-base.gmail` — sends weekly roundup email.
- **Configuration choices:**
  - Recipient: `your_email_id`
  - Subject: `{{$json.subject}}`
  - Message body: `{{$json.htmlEmail}}`
- **Key expressions or variables used:**
  - `={{ $json.htmlEmail }}`
  - `={{ $json.subject }}`
- **Input and output connections:** Input from **Parse & Build Weekly Email**. No downstream connection.
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Gmail auth/sending quota issues
  - Invalid recipient
  - Email client rendering differences
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 30 Min Trigger | n8n-nodes-base.scheduleTrigger | Triggers match polling every 30 minutes |  | Fetch Recent Matches | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| Fetch Recent Matches | n8n-nodes-base.httpRequest | Calls CricAPI for current/recent matches | Every 30 Min Trigger | Read Match Log | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| Read Match Log | n8n-nodes-base.googleSheets | Reads tracked matches to detect already analyzed entries | Fetch Recent Matches | Find New Completed IPL Match | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| Find New Completed IPL Match | n8n-nodes-base.code | Selects one new completed IPL match and normalizes score data | Read Match Log | IF New Match Found; Build Analysis Prompt | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| IF New Match Found | n8n-nodes-base.if | Stops the logging branch if no new match was found | Find New Completed IPL Match | Save to Match Log | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| Save to Match Log | n8n-nodes-base.googleSheets | Appends newly discovered match to Match Log | IF New Match Found |  | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| Build Analysis Prompt | n8n-nodes-base.code | Builds structured prompt for post-match AI analysis | Find New Completed IPL Match | Match Analyst | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Match Analyst | @n8n/n8n-nodes-langchain.openAi | Generates post-match analysis JSON | Build Analysis Prompt | Parse Analysis & Build Email | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Parse Analysis & Build Email | n8n-nodes-base.code | Parses AI JSON and builds HTML post-match email | Match Analyst | Send Analysis Email | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Send Analysis Email | n8n-nodes-base.gmail | Sends post-match analysis email | Parse Analysis & Build Email | Update Match Log | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Update Match Log | n8n-nodes-base.googleSheets | Marks analyzed match as processed and emailed | Send Analysis Email | Log to Analysis Log | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Log to Analysis Log | n8n-nodes-base.googleSheets | Stores summary data for weekly digest generation | Update Match Log |  | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Weekly Monday 9AM Trigger | n8n-nodes-base.scheduleTrigger | Starts weekly roundup every Monday at 9 AM |  | Read Analysis Log | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| Read Analysis Log | n8n-nodes-base.googleSheets | Reads stored post-match analyses | Weekly Monday 9AM Trigger | Build Weekly Digest Prompt | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| Build Weekly Digest Prompt | n8n-nodes-base.code | Filters last 7 days and creates weekly AI prompt | Read Analysis Log | IF Matches This Week | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| IF Matches This Week | n8n-nodes-base.if | Prevents weekly generation when no recent analyses exist | Build Weekly Digest Prompt | Weekly Journalist | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| Weekly Journalist | @n8n/n8n-nodes-langchain.openAi | Generates weekly roundup JSON | IF Matches This Week | Parse & Build Weekly Email | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| Parse & Build Weekly Email | n8n-nodes-base.code | Parses weekly JSON and builds HTML roundup email | Weekly Journalist | Send Weekly Digest | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| Send Weekly Digest | n8n-nodes-base.gmail | Sends weekly roundup email | Parse & Build Weekly Email |  | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## Workflow Overview  \n### How It Works  \nEvery IPL match tells a story that the scoreboard alone cannot capture. This workflow automatically detects completed IPL matches, fetches the full scorecard, and sends it to GPT-4o which produces a detailed journalist-style analysis covering innings breakdowns, tactical decisions, key moments, player of the match, and what the result means for both teams going forward. Every Monday it also generates a weekly roundup digest summarizing all the week's matches in one beautifully designed email.  \n### Setup Steps  \n1. Sign up at cricapi.com and get your API key  \n2. Add CRICAPI KEY in http nodes  \n3. Create the Google Sheet with 2 sheets — Match Log and Analysis Log  \n4. Connect Google Sheets, OpenAI, and Gmail credentials  \n5. Add your email to both Gmail nodes  \n6. Activate the workflow. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | Match Completion Detector runs every 30 minutes and polls CricAPI for recently completed T20 matches. It reads the Match Log sheet to check which matches have already been analyzed and finds any new completed IPL match that hasn't been processed yet. When a new match is found it saves it to the Match Log with analyzed set to false and passes all match data forward to the analysis engine. If no new match is found the workflow stops cleanly. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | This flow receives the completed match data and builds a structured analytical prompt containing both innings stats, run rates, and result context. GPT-4o acts as a cricket analyst and produces a headline, match summary, innings analysis, key moments, tactical breakdown, and player of the match. The response is assembled into a branded HTML email and sent immediately. Both the Match Log and Analysis Log sheets are updated after every send. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | Weekly Digest runs every Monday at 9AM. It reads all match analyses from the past 7 days in the Analysis Log sheet and sends them to GPT-4o which generates a weekly roundup — a headline, match recaps, player of the week, the biggest talking point, and a preview of the week ahead. The digest is sent as a branded HTML email. If no matches were analyzed in the past 7 days the workflow stops cleanly without sending anything. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets file**
   1. Create one spreadsheet, for example: `IPL Post Match Analyzer`.
   2. Add a sheet named **Match Log** with columns:
      - `match_id`
      - `match_name`
      - `date`
      - `winner`
      - `analyzed`
      - `analysis_sent`
   3. Add a second sheet named **Analysis Log** with columns:
      - `date`
      - `match_name`
      - `winner`
      - `player_of_match`
      - `analysis_summary`
      - `email_sent`

2. **Prepare credentials in n8n**
   1. Create **Google Sheets OAuth2** credentials with access to the spreadsheet.
   2. Create **OpenAI API** credentials.
   3. Create **Gmail OAuth2** credentials for the sender mailbox.
   4. Confirm all credentials are authorized and testable.

3. **Create the first trigger**
   1. Add a **Schedule Trigger** node named **Every 30 Min Trigger**.
   2. Set it to run every **30 minutes**.

4. **Create the CricAPI request**
   1. Add an **HTTP Request** node named **Fetch Recent Matches**.
   2. Set method to GET.
   3. Set URL to:
      - `https://api.cricapi.com/v1/currentMatches`
   4. Enable query parameters:
      - `apikey` = your CricAPI key
      - `offset` = `0`
   5. Connect **Every 30 Min Trigger → Fetch Recent Matches**.

5. **Read the match tracking sheet**
   1. Add a **Google Sheets** node named **Read Match Log**.
   2. Select the spreadsheet.
   3. Select the **Match Log** sheet.
   4. Keep read defaults; enable output even when no rows exist if available.
   5. Connect **Fetch Recent Matches → Read Match Log**.

6. **Add the match detection code**
   1. Add a **Code** node named **Find New Completed IPL Match**.
   2. Paste logic equivalent to:
      - read CricAPI result from **Fetch Recent Matches**
      - read all rows from **Read Match Log**
      - build a set of analyzed match IDs
      - find first completed IPL T20 match not yet analyzed
      - if none exists, return `{ skip: true, reason: 'No new completed matches' }`
      - otherwise output:
        - `matchId`
        - `matchName`
        - `matchDate`
        - `teams`
        - `team1`
        - `team2`
        - `status`
        - `innings1Score`
        - `innings2Score`
        - `innings1Runs`
        - `innings1Wickets`
        - `innings1Overs`
        - `innings2Runs`
        - `innings2Wickets`
        - `innings2Overs`
        - `isLiveIPL`
   3. Include the fallback selection for any completed T20 only if you want test-mode behavior.
   4. Connect **Read Match Log → Find New Completed IPL Match**.

7. **Gate the logging branch**
   1. Add an **IF** node named **IF New Match Found**.
   2. Condition:
      - left value: `{{$json.skip}}`
      - operator: `not equals`
      - right value: `true`
   3. Enable loose type validation if needed.
   4. Connect **Find New Completed IPL Match → IF New Match Found**.

8. **Append newly found matches to Match Log**
   1. Add a **Google Sheets** node named **Save to Match Log**.
   2. Set operation to **Append**.
   3. Target the **Match Log** sheet.
   4. Map columns:
      - `match_id` = `{{$json.matchId}}`
      - `match_name` = `{{$json.matchName}}`
      - `date` = `{{$json.matchDate}}`
      - `winner` = `{{$json.status}}`
      - `analyzed` = `false`
      - `analysis_sent` = `false`
   5. Connect the **true** output of **IF New Match Found → Save to Match Log**.

9. **Build the post-match prompt**
   1. Add a **Code** node named **Build Analysis Prompt**.
   2. Read normalized data from **Find New Completed IPL Match**.
   3. Build a long-form prompt that includes:
      - match name
      - result
      - both innings score strings
      - runs, wickets, overs
      - computed run rates
   4. Request strict JSON with fields:
      - `headline`
      - `match_summary`
      - `innings1_analysis`
      - `innings2_analysis`
      - `key_moments`
      - `tactical_breakdown`
      - `player_of_match`
      - `what_to_watch_next`
      - `analysis_summary`
   5. Connect **Find New Completed IPL Match → Build Analysis Prompt**.

10. **Important reproduction note**
    - In the original workflow, **Build Analysis Prompt** is connected directly from **Find New Completed IPL Match**, not from the IF node.  
    - That means the analysis branch may still execute when `skip: true` is returned.  
    - For a safer rebuild, connect **Build Analysis Prompt** from the **true** branch of **IF New Match Found** instead.  
    - If you want exact replication, keep the original direct connection.

11. **Add the post-match OpenAI node**
    1. Add an **OpenAI** node from the LangChain/OpenAI package named **Match Analyst**.
    2. Select model **gpt-4o-mini**.
    3. Pass `{{$json.prompt}}` as the user content.
    4. Optionally add a system message instructing valid JSON only; the original node does not include one.
    5. Connect **Build Analysis Prompt → Match Analyst**.

12. **Parse AI output and build the post-match HTML**
    1. Add a **Code** node named **Parse Analysis & Build Email**.
    2. Read the OpenAI output text from the response structure.
    3. Strip markdown code fences if present.
    4. Parse the JSON.
    5. Read original match context from **Build Analysis Prompt**.
    6. Build an HTML email containing:
       - title banner
       - match/date
       - side-by-side scores
       - headline
       - match summary
       - 1st innings analysis
       - 2nd innings analysis
       - key moments list
       - tactical breakdown
       - player of the match
       - what to watch next
    7. Output:
       - `htmlEmail`
       - `subject`
       - `headline`
       - `match_summary`
       - `player_of_match`
       - `analysis_summary`
       - original match data
    8. Connect **Match Analyst → Parse Analysis & Build Email**.

13. **Send the post-match email**
    1. Add a **Gmail** node named **Send Analysis Email**.
    2. Configure Gmail OAuth2 credentials.
    3. Set:
       - To: your recipient email
       - Subject: `{{$json.subject}}`
       - Message: `{{$json.htmlEmail}}`
    4. Connect **Parse Analysis & Build Email → Send Analysis Email**.

14. **Mark the match as analyzed**
    1. Add a **Google Sheets** node named **Update Match Log**.
    2. Set operation to **Update**.
    3. Use **Match Log** sheet.
    4. Set matching column to `match_id`.
    5. Map:
       - `match_id` = `{{$('Parse Analysis & Build Email').item.json.matchId}}`
       - `analyzed` = `true`
       - `analysis_sent` = `true`
    6. Optionally enable execute once.
    7. Connect **Send Analysis Email → Update Match Log**.

15. **Append to Analysis Log**
    1. Add a **Google Sheets** node named **Log to Analysis Log**.
    2. Set operation to **Append**.
    3. Use **Analysis Log** sheet.
    4. Map:
       - `date` = `{{ new Date().toISOString().split('T')[0] }}`
       - `match_name` = `{{$('Parse Analysis & Build Email').item.json.matchName}}`
       - `winner` = `{{$('Parse Analysis & Build Email').item.json.status}}`
       - `player_of_match` = `{{$('Parse Analysis & Build Email').item.json.player_of_match}}`
       - `analysis_summary` = `{{$('Parse Analysis & Build Email').item.json.analysis_summary}}`
       - `email_sent` = `true`
    5. Connect **Update Match Log → Log to Analysis Log**.

16. **Create the weekly trigger**
    1. Add another **Schedule Trigger** node named **Weekly Monday 9AM Trigger**.
    2. Configure it for:
       - every week
       - Monday
       - 9:00 AM

17. **Read the analysis history**
    1. Add a **Google Sheets** node named **Read Analysis Log**.
    2. Select the same spreadsheet.
    3. Use the **Analysis Log** sheet.
    4. Connect **Weekly Monday 9AM Trigger → Read Analysis Log**.

18. **Build the weekly digest prompt**
    1. Add a **Code** node named **Build Weekly Digest Prompt**.
    2. Read all rows from input.
    3. If zero rows exist, return:
       - `skip: true`
       - `reason: 'No analyses this week'`
    4. Compute the date 7 days ago.
    5. Filter rows where `date >= sevenDaysAgo`.
    6. If no rows remain, return:
       - `skip: true`
       - `reason: 'No matches this week'`
    7. Build a prompt summarizing each match:
       - match name
       - result
       - player of match
       - analysis summary
    8. Ask for strict JSON with:
       - `week_headline`
       - `week_intro`
       - `match_recaps`
       - `player_of_week`
       - `week_talking_point`
       - `next_week_preview`
    9. Output:
       - `prompt`
       - `matchCount`
       - `weekOf`
    10. Connect **Read Analysis Log → Build Weekly Digest Prompt**.

19. **Gate weekly generation**
    1. Add an **IF** node named **IF Matches This Week**.
    2. Condition:
       - `{{$json.skip}} != true`
    3. Connect **Build Weekly Digest Prompt → IF Matches This Week**.

20. **Generate weekly roundup with OpenAI**
    1. Add an **OpenAI** node named **Weekly Journalist**.
    2. Model: **gpt-4o-mini**.
    3. User content: `{{$json.prompt}}`
    4. Add system message:
       - “You are an expert IPL cricket journalist. Always respond in valid JSON only. No markdown, no code blocks.”
    5. Connect **IF Matches This Week** true branch → **Weekly Journalist**.

21. **Parse the weekly digest and build HTML**
    1. Add a **Code** node named **Parse & Build Weekly Email**.
    2. Extract and clean the model response text.
    3. Parse JSON.
    4. Read context from **Build Weekly Digest Prompt**.
    5. Build HTML containing:
       - roundup header
       - week headline
       - intro
       - match recap cards
       - player of the week
       - talking point
       - next week preview
    6. Output:
       - `htmlEmail`
       - `subject`
       - `matchCount`
       - `weekOf`
    7. Connect **Weekly Journalist → Parse & Build Weekly Email**.

22. **Send the weekly digest**
    1. Add a **Gmail** node named **Send Weekly Digest**.
    2. Configure Gmail OAuth2 credentials.
    3. Set:
       - To: your email
       - Subject: `{{$json.subject}}`
       - Message: `{{$json.htmlEmail}}`
    4. Connect **Parse & Build Weekly Email → Send Weekly Digest**.

23. **Optional canvas notes**
    1. Add sticky notes documenting:
       - overall workflow overview and setup steps
       - match completion detector explanation
       - post-match analysis explanation
       - weekly digest explanation

24. **Test the workflow**
    1. Run the post-match branch manually.
    2. Confirm CricAPI returns data.
    3. Confirm the Google sheet is readable and writable.
    4. Confirm OpenAI returns valid JSON.
    5. Confirm Gmail sends HTML correctly.
    6. Then test the weekly branch using sample rows in **Analysis Log**.

25. **Activate the workflow**
    1. Replace placeholders:
       - `your_api_key`
       - `your_email_id`
    2. Activate the workflow so both schedule triggers run automatically.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview note explains the business purpose: detect completed IPL matches, generate journalist-style AI analysis, and send weekly digests. | Canvas documentation |
| Setup note says to sign up at CricAPI, add the API key to HTTP nodes, create two Google Sheets tabs, connect Google Sheets/OpenAI/Gmail credentials, add recipient email addresses, and activate the workflow. | CricAPI: https://cricapi.com/ |
| Match detection note explains the polling logic, duplicate prevention using Match Log, and clean stop behavior when no new match is available. | Canvas documentation |
| Post-match analysis note explains GPT-powered tactical breakdown generation, HTML email assembly, and post-send updates to Match Log and Analysis Log. | Canvas documentation |
| Weekly digest note explains Monday 9 AM scheduling, reading the last 7 days of analyses, generating roundup content, and stopping cleanly when there are no recent matches. | Canvas documentation |