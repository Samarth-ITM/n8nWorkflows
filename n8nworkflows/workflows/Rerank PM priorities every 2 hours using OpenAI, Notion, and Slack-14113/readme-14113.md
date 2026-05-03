Rerank PM priorities every 2 hours using OpenAI, Notion, and Slack

https://n8nworkflows.xyz/workflows/rerank-pm-priorities-every-2-hours-using-openai--notion--and-slack-14113


# Rerank PM priorities every 2 hours using OpenAI, Notion, and Slack

# 1. Workflow Overview

This workflow periodically re-ranks product-management priorities by combining operational context from multiple Notion databases, processing that context through three OpenAI scoring passes, and then updating Notion and Slack only when a meaningful ranking result is produced.

Its main use case is continuous prioritization during working hours: every 2 hours on weekdays, it gathers open priorities plus fresh contextual signals such as impactful events, meeting decisions, urgent emails, and blocked actions. It then builds a derived priority matrix, asks AI to score impact and urgency, generates a final ranking, compares the outcome, and posts/logs the result.

## 1.1 Scheduled Trigger and Runtime Gate
The workflow starts on a workday schedule, reads a settings database in Notion, and checks a demo-mode flag before continuing.

## 1.2 Context Collection
Five parallel branches collect and normalize:
- open priorities
- recent signals
- meeting decisions
- urgent emails
- blocked actions

## 1.3 Priority Matrix Construction
The normalized datasets are merged into one combined context object, then transformed into a matrix linking each priority with related signals, emails, and blockers.

## 1.4 Three-Pass AI Scoring
The workflow sends the enriched context through three OpenAI chat passes:
1. impact analysis
2. urgency recalibration
3. final ranking generation

Each pass is parsed from model output into machine-usable JSON.

## 1.5 Change Handling, Notifications, and Audit
The final ranking is checked for changes, then:
- if ranking data exists, a Slack summary is posted and priorities are batch-updated in Notion
- if not, a “no changes” Slack message is sent
- both paths end in an audit log written to Notion

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Runtime Gate

**Overview:**  
This block controls when the workflow runs and whether the main logic should proceed. It uses a recurring weekday schedule and a Notion settings read to determine whether demo mode is enabled.

**Nodes Involved:**  
- Rerank Trigger
- Read Settings
- Check Demo Mode

### Rerank Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Uses a cron expression: `0 */2 8-18 * * 1-5`.
  - Runs every 2 hours
  - At minute 0
  - Between 08:00 and 18:00
  - Monday to Friday
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none
  - Output: `Read Settings`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Cron timezone may differ from business expectations depending on workflow/server timezone.
  - If the workflow is inactive, it will not run.
- **Sub-workflow reference:** None.

### Read Settings
- **Type and technical role:** `n8n-nodes-base.notion`; fetches configuration pages from a Notion database.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Return all: enabled
  - Database ID: `31e06bab-3ebe-813a-858d-dfec8df6b644`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Rerank Trigger`
  - Output: `Check Demo Mode`
- **Version-specific requirements:** Type version `2.2`; requires valid Notion credentials and access to the settings database.
- **Edge cases or potential failure types:**  
  - Notion auth failure
  - Database not found / insufficient permissions
  - Multiple returned settings pages may create multiple downstream executions
- **Sub-workflow reference:** None.

### Check Demo Mode
- **Type and technical role:** `n8n-nodes-base.if`; conditional gate based on a Notion checkbox property.
- **Configuration choices:**  
  - Condition checks whether `{{$json.properties?.demo_mode?.checkbox}}` equals `true`
- **Key expressions or variables used:**  
  - `{{$json.properties?.demo_mode?.checkbox}}`
- **Input and output connections:**  
  - Input: `Read Settings`
  - True output:  
    - `Get Open Priorities`
    - `Get Recent Signals`
    - `Get Meeting Decisions`
    - `Get Urgent Emails`
    - `Get Blocked Actions`
  - False output: not connected
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If `demo_mode` is missing, the expression resolves to `undefined`, so the condition fails.
  - Because only the true branch is connected, the workflow silently stops when demo mode is false.
  - If multiple settings pages are returned, each with different values, downstream behavior may duplicate or fragment execution.
- **Sub-workflow reference:** None.

---

## 2.2 Context Collection

**Overview:**  
This block gathers all source data used for re-ranking. It runs five Notion retrieval branches in parallel and normalizes each source into a simplified JSON structure for later merging.

**Nodes Involved:**  
- Get Open Priorities
- Sort Priorities
- Limit Top 20
- Normalize Priorities
- Get Recent Signals
- Filter High Signals
- Normalize Signals
- Get Meeting Decisions
- Normalize Meetings
- Get Urgent Emails
- Filter Urgent Emails
- Normalize Emails
- Get Blocked Actions
- Filter Overdue Actions
- Normalize Blockers

### Get Open Priorities
- **Type and technical role:** `n8n-nodes-base.notion`; retrieves candidate priorities from Notion.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Return all: enabled
  - Database ID: `31e06bab-3ebe-81fe-9d0c-e65a49a99ef3`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Check Demo Mode`
  - Output: `Sort Priorities`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - No filter is applied here despite the node name implying “open priorities”; closed items may still be returned unless the database view itself enforces it.
  - Notion pagination/load may be significant if the database is large.
- **Sub-workflow reference:** None.

### Sort Priorities
- **Type and technical role:** `n8n-nodes-base.sort`; orders the priority items before limiting.
- **Configuration choices:**  
  - No explicit sort fields are configured in the exported JSON.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Get Open Priorities`
  - Output: `Limit Top 20`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - With no explicit sort rules, resulting order may default to input order and may not be deterministic enough for “current rank.”
- **Sub-workflow reference:** None.

### Limit Top 20
- **Type and technical role:** `n8n-nodes-base.limit`; keeps only the first 20 items.
- **Configuration choices:**  
  - `maxItems = 20`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Sort Priorities`
  - Output: `Normalize Priorities`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - If sorting is undefined, the selected top 20 may not reflect actual business priority.
- **Sub-workflow reference:** None.

### Normalize Priorities
- **Type and technical role:** `n8n-nodes-base.code`; maps Notion pages to a clean `priorities` array.
- **Configuration choices:**  
  - Produces one output item:
    - `source: 'priorities'`
    - `items: [...]`
  - For each priority it extracts:
    - `id`
    - `title`
    - `currentRank` as array index + 1
    - `status`
    - `priority`
    - `due`
- **Key expressions or variables used:**  
  - Uses `$input.all()`
  - Property paths such as:
    - `item.properties?.Name?.title?.[0]?.plain_text`
    - `item.properties?.Status?.select?.name`
    - `item.properties?.Priority?.number`
    - `item.properties?.Due?.date?.start`
- **Input and output connections:**  
  - Input: `Limit Top 20`
  - Output: `Merge Context`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing title becomes `Untitled`
  - Missing optional fields default to `Open`, `0`, or `null`
  - Current rank depends entirely on previous node order
- **Sub-workflow reference:** None.

### Get Recent Signals
- **Type and technical role:** `n8n-nodes-base.notion`; retrieves signal records.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Return all: enabled
  - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Check Demo Mode`
  - Output: `Filter High Signals`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** Notion auth/access failures, large result sets.
- **Sub-workflow reference:** None.

### Filter High Signals
- **Type and technical role:** `n8n-nodes-base.filter`; keeps signals that have an impact value.
- **Configuration choices:**  
  - Condition: `{{$json.properties?.Impact?.select?.name}}` is not empty
- **Key expressions or variables used:**  
  - `{{$json.properties?.Impact?.select?.name}}`
- **Input and output connections:**  
  - Input: `Get Recent Signals`
  - Output: `Normalize Signals`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - This checks non-empty impact, not specifically “high” impact despite the node name.
- **Sub-workflow reference:** None.

### Normalize Signals
- **Type and technical role:** `n8n-nodes-base.code`; converts signals into a clean normalized structure.
- **Configuration choices:**  
  - Produces:
    - `source: 'signals'`
    - `items: [{ id, title, impact, category }]`
- **Key expressions or variables used:**  
  - `i.properties?.Name?.title?.[0]?.plain_text`
  - `i.properties?.Impact?.select?.name || 'Medium'`
  - `i.properties?.Category?.select?.name || 'General'`
- **Input and output connections:**  
  - Input: `Filter High Signals`
  - Output: `Merge Context`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Defaults may hide incomplete data.
- **Sub-workflow reference:** None.

### Get Meeting Decisions
- **Type and technical role:** `n8n-nodes-base.notion`; retrieves meeting-related decision records.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Return all: enabled
  - Database ID: `31e06bab-3ebe-8108-b9d7-cde4bd8192b6`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Check Demo Mode`
  - Output: `Normalize Meetings`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** Same general Notion issues.
- **Sub-workflow reference:** None.

### Normalize Meetings
- **Type and technical role:** `n8n-nodes-base.code`; compresses meeting pages into a simplified array.
- **Configuration choices:**  
  - Produces:
    - `source: 'meetings'`
    - `items: [{ id, title, decisions }]`
  - Rich text decisions are concatenated into a single string.
- **Key expressions or variables used:**  
  - `i.properties?.Decisions?.rich_text?.map(t => t.plain_text).join(' ')`
- **Input and output connections:**  
  - Input: `Get Meeting Decisions`
  - Output: `Merge Context`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Empty decision text becomes empty string
  - Loses structure/formatting from Notion rich text
- **Sub-workflow reference:** None.

### Get Urgent Emails
- **Type and technical role:** `n8n-nodes-base.notion`; retrieves email records stored in Notion.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Return all: enabled
  - Database ID: `31e06bab-3ebe-81d6-8c12-f607c8ff3b3f`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Check Demo Mode`
  - Output: `Filter Urgent Emails`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** Same general Notion issues.
- **Sub-workflow reference:** None.

### Filter Urgent Emails
- **Type and technical role:** `n8n-nodes-base.filter`; restricts email records to urgent ones.
- **Configuration choices:**  
  - Condition: `{{$json.properties?.Priority?.select?.name}} == 'Urgent'`
- **Key expressions or variables used:**  
  - `{{$json.properties?.Priority?.select?.name}}`
- **Input and output connections:**  
  - Input: `Get Urgent Emails`
  - Output: `Normalize Emails`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Case-sensitive exact match; values like `urgent` or `High` will not pass.
- **Sub-workflow reference:** None.

### Normalize Emails
- **Type and technical role:** `n8n-nodes-base.code`; converts email pages into a simplified structure.
- **Configuration choices:**  
  - Produces:
    - `source: 'emails'`
    - `items: [{ id, subject, sender, urgency }]`
- **Key expressions or variables used:**  
  - `Subject.title[0].plain_text`
  - `From.email`
  - `Priority.select.name`
- **Input and output connections:**  
  - Input: `Filter Urgent Emails`
  - Output: `Merge Context`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - `From.email` assumes the Notion property is an email field; if it is people/text, it returns empty.
- **Sub-workflow reference:** None.

### Get Blocked Actions
- **Type and technical role:** `n8n-nodes-base.notion`; fetches action items or tasks.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Return all: enabled
  - Database ID: `31e06bab-3ebe-8190-894f-f211aea15dc4`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Check Demo Mode`
  - Output: `Filter Overdue Actions`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** Same general Notion issues.
- **Sub-workflow reference:** None.

### Filter Overdue Actions
- **Type and technical role:** `n8n-nodes-base.filter`; keeps only blocked actions.
- **Configuration choices:**  
  - Condition: `{{$json.properties?.Status?.select?.name}} == 'Blocked'`
- **Key expressions or variables used:**  
  - `{{$json.properties?.Status?.select?.name}}`
- **Input and output connections:**  
  - Input: `Get Blocked Actions`
  - Output: `Normalize Blockers`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Despite the node name, it does not filter overdue items, only blocked ones.
- **Sub-workflow reference:** None.

### Normalize Blockers
- **Type and technical role:** `n8n-nodes-base.code`; normalizes blocked action records.
- **Configuration choices:**  
  - Produces:
    - `source: 'blockers'`
    - `items: [{ id, title, status, due }]`
- **Key expressions or variables used:**  
  - `Name.title[0].plain_text`
  - `Status.select.name`
  - `Due.date.start`
- **Input and output connections:**  
  - Input: `Filter Overdue Actions`
  - Output: `Merge Context`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Missing values fall back to `Open` and `null`.
- **Sub-workflow reference:** None.

---

## 2.3 Priority Matrix Construction

**Overview:**  
This block merges all normalized branches and creates an enriched matrix for each priority. It attempts lightweight entity matching by comparing words in priority titles with text in signals, email subjects, and blocker titles.

**Nodes Involved:**  
- Merge Context
- Build Priority Matrix
- Remove Duplicates

### Merge Context
- **Type and technical role:** `n8n-nodes-base.merge`; combines multiple branch outputs.
- **Configuration choices:**  
  - Mode: `combine`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Inputs:
    - `Normalize Priorities`
    - `Normalize Signals`
    - `Normalize Meetings`
    - `Normalize Emails`
    - `Normalize Blockers`
  - Output: `Build Priority Matrix`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Combine mode behavior depends on n8n merge semantics; if one branch is empty or delayed, output shape may differ from expectations.
  - This node is central to data integrity; malformed normalized outputs can break matrix generation.
- **Sub-workflow reference:** None.

### Build Priority Matrix
- **Type and technical role:** `n8n-nodes-base.code`; assembles all source arrays and enriches priorities with related context.
- **Configuration choices:**  
  - Builds a `context` object with arrays:
    - `priorities`
    - `signals`
    - `meetings`
    - `emails`
    - `blockers`
  - For each priority, calculates:
    - `relatedSignals`
    - `relatedEmails`
    - `relatedBlockers`
    - `signalCount`
    - `emailCount`
    - `blockerCount`
  - Also returns workflow-level totals.
- **Key expressions or variables used:**  
  - Uses `$input.all()`
  - Matching heuristic:
    - lowercases titles
    - splits priority title into words
    - matches only words longer than 3 characters
    - checks substring inclusion
- **Input and output connections:**  
  - Input: `Merge Context`
  - Output: `Remove Duplicates`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Matching is heuristic and may produce false positives/negatives.
  - Meetings are included globally but not explicitly linked per priority.
  - Common words longer than 3 characters may create noisy associations.
- **Sub-workflow reference:** None.

### Remove Duplicates
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`; attempts to deduplicate output before AI processing.
- **Configuration choices:**  
  - No specific fields configured in the JSON.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Build Priority Matrix`
  - Output: `AI Pass 1 - Impact`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Because `Build Priority Matrix` already emits a single aggregated item, this node may be redundant.
  - Without explicit field settings, deduplication behavior may be ambiguous.
- **Sub-workflow reference:** None.

---

## 2.4 Three-Pass AI Scoring

**Overview:**  
This block sends the enriched context through three sequential OpenAI chat calls. Each pass extracts structured JSON from the model response and stores it under a named field for downstream ranking logic.

**Nodes Involved:**  
- AI Pass 1 - Impact
- Parse Impact
- Set Impact Scores
- AI Pass 2 - Urgency
- Parse Urgency
- Set Urgency Scores
- AI Pass 3 - Final Rank
- Parse Ranking

### AI Pass 1 - Impact
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; OpenAI chat model call for impact scoring.
- **Configuration choices:**  
  - Resource: `chat`
  - No prompt details are included in the JSON export.
- **Key expressions or variables used:** Not visible in export.
- **Input and output connections:**  
  - Input: `Remove Duplicates`
  - Output: `Parse Impact`
- **Version-specific requirements:** Type version `1.4`; requires OpenAI credentials and compatible node package availability.
- **Edge cases or potential failure types:**  
  - Missing credentials
  - Rate limiting
  - Timeout
  - Non-JSON or malformed natural-language output
  - Hallucinated schema mismatch
- **Sub-workflow reference:** None.

### Parse Impact
- **Type and technical role:** `n8n-nodes-base.code`; extracts JSON from model output.
- **Configuration choices:**  
  - Reads `message.content`
  - Uses regex to find the first JSON object
  - Parses into:
    - `impactScores: parsed.impactScores || parsed`
- **Key expressions or variables used:**  
  - `$json.message?.content`
  - `/\{[\s\S]*\}/`
- **Input and output connections:**  
  - Input: `AI Pass 1 - Impact`
  - Output: `Set Impact Scores`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If the model returns an array root instead of object root, parsing logic may fail to capture it.
  - Regex can overmatch if the content contains multiple brace blocks.
  - On parse failure, it silently falls back to empty scores.
- **Sub-workflow reference:** None.

### Set Impact Scores
- **Type and technical role:** `n8n-nodes-base.set`; stores parsed impact data under a predictable field.
- **Configuration choices:**  
  - Assigns:
    - `impact = {{$json}}`
- **Key expressions or variables used:**  
  - `{{$json}}`
- **Input and output connections:**  
  - Input: `Parse Impact`
  - Output: `AI Pass 2 - Urgency`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - This wraps current JSON rather than merging with the original matrix unless prior execution data is preserved elsewhere; depending on intended design, contextual data may be lost.
- **Sub-workflow reference:** None.

### AI Pass 2 - Urgency
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; second chat pass for urgency scoring.
- **Configuration choices:**  
  - Resource: `chat`
  - Prompt/model parameters are not visible in export.
- **Key expressions or variables used:** Not visible.
- **Input and output connections:**  
  - Input: `Set Impact Scores`
  - Output: `Parse Urgency`
- **Version-specific requirements:** Type version `1.4`; OpenAI credentials required.
- **Edge cases or potential failure types:** Same as pass 1, plus schema drift between passes.
- **Sub-workflow reference:** None.

### Parse Urgency
- **Type and technical role:** `n8n-nodes-base.code`; extracts urgency JSON from the model response.
- **Configuration choices:**  
  - Outputs:
    - `urgencyScores: parsed.urgencyScores || parsed`
- **Key expressions or variables used:**  
  - `$json.message?.content`
- **Input and output connections:**  
  - Input: `AI Pass 2 - Urgency`
  - Output: `Set Urgency Scores`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Same parsing fragility as `Parse Impact`.
- **Sub-workflow reference:** None.

### Set Urgency Scores
- **Type and technical role:** `n8n-nodes-base.set`; stores urgency results under `urgency`.
- **Configuration choices:**  
  - Assigns:
    - `urgency = {{$json}}`
- **Key expressions or variables used:**  
  - `{{$json}}`
- **Input and output connections:**  
  - Input: `Parse Urgency`
  - Output: `AI Pass 3 - Final Rank`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Like the previous Set node, it may narrow data to only current parsed content if not intentionally merged.
- **Sub-workflow reference:** None.

### AI Pass 3 - Final Rank
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; final AI ranking pass.
- **Configuration choices:**  
  - Resource: `chat`
  - Prompt/model settings not present in the export.
- **Key expressions or variables used:** Not visible.
- **Input and output connections:**  
  - Input: `Set Urgency Scores`
  - Output: `Parse Ranking`
- **Version-specific requirements:** Type version `1.4`.
- **Edge cases or potential failure types:**  
  - If previous context was lost, the final pass may lack enough information to build valid rankings.
  - Output schema must include ranking-compatible objects expected downstream.
- **Sub-workflow reference:** None.

### Parse Ranking
- **Type and technical role:** `n8n-nodes-base.code`; parses final model output into `newRanking`.
- **Configuration choices:**  
  - Outputs:
    - `newRanking: parsed.ranking || []`
- **Key expressions or variables used:**  
  - `$json.message?.content`
- **Input and output connections:**  
  - Input: `AI Pass 3 - Final Rank`
  - Output: `Compare Rankings`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Silent fallback to empty ranking can route execution into “no changes” even when the model actually failed.
- **Sub-workflow reference:** None.

---

## 2.5 Change Handling, Notifications, and Audit

**Overview:**  
This block evaluates the final ranking output, formats user-facing summaries, updates Notion pages in batches, notifies Slack, and records an audit event. The actual branching currently checks whether ranking data exists rather than whether the compare step found true differences.

**Nodes Involved:**  
- Compare Rankings
- Any Changes
- Build Change Summary
- Prepare Update Batch
- Batch Updates
- Update Notion Priority
- Format Slack Update
- Post Ranking Change
- No Changes
- Slack No Changes
- Create Audit Log

### Compare Rankings
- **Type and technical role:** `n8n-nodes-base.compareDatasets`; intended to compare old vs new ranking records.
- **Configuration choices:**  
  - Merge by fields:
    - `id` ↔ `id`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Parse Ranking`
  - Output: `Any Changes`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Only one input connection is visible in the workflow, so this node may not be comparing two datasets as intended.
  - If misconfigured, downstream change detection is effectively bypassed.
- **Sub-workflow reference:** None.

### Any Changes
- **Type and technical role:** `n8n-nodes-base.if`; branch controller for update vs no-op path.
- **Configuration choices:**  
  - Condition compares string value of:
    - `{{$json.newRanking?.length > 0}}`
    - against `"true"`
- **Key expressions or variables used:**  
  - `{{$json.newRanking?.length > 0}}`
- **Input and output connections:**  
  - Input: `Compare Rankings`
  - True output:
    - `Build Change Summary`
    - `Prepare Update Batch`
  - False output:
    - `No Changes`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - This does not verify whether positions actually changed; it only checks whether any ranking items exist.
  - A valid but unchanged ranking still takes the “changes” path.
- **Sub-workflow reference:** None.

### Build Change Summary
- **Type and technical role:** `n8n-nodes-base.code`; creates a detailed text summary of the ranking.
- **Configuration choices:**  
  - Builds `changeSummary`
  - Builds `changedCount`
  - Expects ranking objects with:
    - `rank`
    - `change`
    - `title`
    - `finalScore`
    - `rationale`
- **Key expressions or variables used:**  
  - `$json.newRanking || []`
  - JavaScript `new Date()`
- **Input and output connections:**  
  - Input: `Any Changes`
  - Output: `Format Slack Update`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If ranking entries lack `change`, arrows default to white-circle behavior.
  - Dates are generated in server timezone/locale behavior.
- **Sub-workflow reference:** None.

### Prepare Update Batch
- **Type and technical role:** `n8n-nodes-base.code`; converts ranking array into one item per Notion page update.
- **Configuration choices:**  
  - Maps each ranking row to:
    - `pageId`
    - `rank`
    - `finalScore`
    - `rationale`
- **Key expressions or variables used:**  
  - `$json.newRanking || []`
- **Input and output connections:**  
  - Input: `Any Changes`
  - Output: `Batch Updates`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If ranking array is empty, this branch yields no update items.
  - Assumes every ranking item includes a valid Notion page ID.
- **Sub-workflow reference:** None.

### Batch Updates
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; loops through update items.
- **Configuration choices:**  
  - Default batching behavior; no explicit batch size configured
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Prepare Update Batch`
  - Output:
    - `Update Notion Priority`
    - loops back from `Update Notion Priority`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Without explicit batch size, behavior relies on node defaults.
  - Loop misconfiguration can stall or overrun if update node errors.
- **Sub-workflow reference:** None.

### Update Notion Priority
- **Type and technical role:** `n8n-nodes-base.notion`; updates a specific Notion page.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `update`
  - `pageId = {{$json.pageId}}`
  - No explicit property updates are present in the exported parameters
- **Key expressions or variables used:**  
  - `{{$json.pageId}}`
- **Input and output connections:**  
  - Input: `Batch Updates`
  - Output: `Batch Updates` loop continuation
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - As exported, this node appears incomplete: it identifies the page but does not specify which properties to update.
  - If left as-is, it may perform no meaningful field changes or fail depending on node validation.
  - Notion permission and schema mismatch risks apply.
- **Sub-workflow reference:** None.

### Format Slack Update
- **Type and technical role:** `n8n-nodes-base.code`; creates a compact Slack-friendly message.
- **Configuration choices:**  
  - Builds `slackText`
  - Includes top 5 ranking items only
- **Key expressions or variables used:**  
  - `$json.newRanking || []`
- **Input and output connections:**  
  - Input: `Build Change Summary`
  - Output: `Post Ranking Change`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If ranking items are malformed, text may contain `undefined`.
- **Sub-workflow reference:** None.

### Post Ranking Change
- **Type and technical role:** `n8n-nodes-base.slack`; sends Slack notification for ranking updates.
- **Configuration choices:**  
  - Text: `{{$json.slackText}}`
- **Key expressions or variables used:**  
  - `{{$json.slackText}}`
- **Input and output connections:**  
  - Input: `Format Slack Update`
  - Output: `Create Audit Log`
- **Version-specific requirements:** Type version `2.2`; requires Slack credentials and channel configuration in the node/credentials layer.
- **Edge cases or potential failure types:**  
  - Slack auth failure
  - Missing target channel configuration
  - Formatting differences depending on Slack app permissions
- **Sub-workflow reference:** None.

### No Changes
- **Type and technical role:** `n8n-nodes-base.noOp`; placeholder branch for no-change flow.
- **Configuration choices:** none.
- **Key expressions or variables used:** none.
- **Input and output connections:**  
  - Input: `Any Changes` false branch
  - Output: `Slack No Changes`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None significant.
- **Sub-workflow reference:** None.

### Slack No Changes
- **Type and technical role:** `n8n-nodes-base.slack`; sends informational Slack message when no ranking output is detected.
- **Configuration choices:**  
  - Text expression:
    - `:white_check_mark: Priority reranker ran at <ISO timestamp> - no ranking changes detected.`
- **Key expressions or variables used:**  
  - `{{ ':white_check_mark: Priority reranker ran at ' + new Date().toISOString() + ' - no ranking changes detected.' }}`
- **Input and output connections:**  
  - Input: `No Changes`
  - Output: `Create Audit Log`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Message text may be misleading because this path can also represent parsing failure or empty AI output, not true “no changes.”
- **Sub-workflow reference:** None.

### Create Audit Log
- **Type and technical role:** `n8n-nodes-base.notion`; creates an audit record in a Notion database.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Database ID: `31e06bab-3ebe-811b-af1f-f186b6dbdf3e`
  - Operation is implicitly page creation into the database
- **Key expressions or variables used:** None visible.
- **Input and output connections:**  
  - Inputs:
    - `Post Ranking Change`
    - `Slack No Changes`
  - Output: none
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - No explicit property mapping is shown in export; audit records may be incomplete unless defaults are configured in-node elsewhere.
  - Notion create permissions required.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Rerank Trigger | Schedule Trigger | Starts the workflow every 2 hours on weekdays during work hours |  | Read Settings | ## How it works Every 2 hours during work days, re-evaluates all open priorities using fresh signals, meetings, emails, and blockers. Three AI passes (Impact, Urgency, Final Ranking) produce a new ranking. Only updates Notion if the ranking actually changed. |
| Read Settings | Notion | Reads workflow settings from Notion | Rerank Trigger | Check Demo Mode | ## How it works Every 2 hours during work days, re-evaluates all open priorities using fresh signals, meetings, emails, and blockers. Three AI passes (Impact, Urgency, Final Ranking) produce a new ranking. Only updates Notion if the ranking actually changed. |
| Check Demo Mode | If | Allows execution to continue only when demo mode is enabled | Read Settings | Get Open Priorities; Get Recent Signals; Get Meeting Decisions; Get Urgent Emails; Get Blocked Actions | ## How it works Every 2 hours during work days, re-evaluates all open priorities using fresh signals, meetings, emails, and blockers. Three AI passes (Impact, Urgency, Final Ranking) produce a new ranking. Only updates Notion if the ranking actually changed. |
| Get Open Priorities | Notion | Retrieves candidate priorities from Notion | Check Demo Mode | Sort Priorities | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Sort Priorities | Sort | Orders priority records before truncation | Get Open Priorities | Limit Top 20 | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Limit Top 20 | Limit | Restricts priority set to 20 items | Sort Priorities | Normalize Priorities | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Normalize Priorities | Code | Converts priority pages into a normalized array | Limit Top 20 | Merge Context | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Get Recent Signals | Notion | Retrieves signal records | Check Demo Mode | Filter High Signals | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Filter High Signals | Filter | Keeps only signals with an impact value | Get Recent Signals | Normalize Signals | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Normalize Signals | Code | Converts signals into a normalized array | Filter High Signals | Merge Context | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Get Meeting Decisions | Notion | Retrieves meeting decision records | Check Demo Mode | Normalize Meetings | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Normalize Meetings | Code | Converts meeting pages into a normalized array | Get Meeting Decisions | Merge Context | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Get Urgent Emails | Notion | Retrieves email records from Notion | Check Demo Mode | Filter Urgent Emails | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Filter Urgent Emails | Filter | Keeps only urgent emails | Get Urgent Emails | Normalize Emails | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Normalize Emails | Code | Converts emails into a normalized array | Filter Urgent Emails | Merge Context | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Get Blocked Actions | Notion | Retrieves action/task records | Check Demo Mode | Filter Overdue Actions | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Filter Overdue Actions | Filter | Keeps blocked action items | Get Blocked Actions | Normalize Blockers | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Normalize Blockers | Code | Converts blockers into a normalized array | Filter Overdue Actions | Merge Context | ## Context Collection Five parallel pulls: open priorities, recent high-impact signals, today's meeting decisions, recent urgent emails, and overdue or blocked actions. |
| Merge Context | Merge | Combines all normalized context branches | Normalize Priorities; Normalize Signals; Normalize Meetings; Normalize Emails; Normalize Blockers | Build Priority Matrix | ## Priority Matrix Builds a detailed context matrix linking each priority to its relevant signals, emails, and decisions. Deduplicates and normalizes the data. |
| Build Priority Matrix | Code | Creates enriched priority matrix with related context counts | Merge Context | Remove Duplicates | ## Priority Matrix Builds a detailed context matrix linking each priority to its relevant signals, emails, and decisions. Deduplicates and normalizes the data. |
| Remove Duplicates | Remove Duplicates | Deduplicates matrix output before AI processing | Build Priority Matrix | AI Pass 1 - Impact | ## Priority Matrix Builds a detailed context matrix linking each priority to its relevant signals, emails, and decisions. Deduplicates and normalizes the data. |
| AI Pass 1 - Impact | OpenAI Chat | First AI pass for impact scoring | Remove Duplicates | Parse Impact | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| Parse Impact | Code | Parses structured impact JSON from model output | AI Pass 1 - Impact | Set Impact Scores | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| Set Impact Scores | Set | Stores impact result under a named field | Parse Impact | AI Pass 2 - Urgency | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| AI Pass 2 - Urgency | OpenAI Chat | Second AI pass for urgency scoring | Set Impact Scores | Parse Urgency | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| Parse Urgency | Code | Parses structured urgency JSON from model output | AI Pass 2 - Urgency | Set Urgency Scores | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| Set Urgency Scores | Set | Stores urgency result under a named field | Parse Urgency | AI Pass 3 - Final Rank | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| AI Pass 3 - Final Rank | OpenAI Chat | Final AI pass producing ranking and rationale | Set Urgency Scores | Parse Ranking | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| Parse Ranking | Code | Parses final ranking JSON from model output | AI Pass 3 - Final Rank | Compare Rankings | ## 3-Pass AI Scoring Pass 1: Impact Analysis scores each priority. Pass 2: Urgency Recalibration adjusts for deadlines. Pass 3: Final Ranking with rationale. |
| Compare Rankings | Compare Datasets | Intended to compare prior and new rankings | Parse Ranking | Any Changes | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Any Changes | If | Routes execution based on presence of ranking output | Compare Rankings | Build Change Summary; Prepare Update Batch; No Changes | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Build Change Summary | Code | Creates detailed reranking summary text | Any Changes | Format Slack Update | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Prepare Update Batch | Code | Expands ranking into one page update item per priority | Any Changes | Batch Updates | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Batch Updates | Split In Batches | Iterates through Notion updates | Prepare Update Batch; Update Notion Priority | Update Notion Priority | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Update Notion Priority | Notion | Updates priority pages in Notion | Batch Updates | Batch Updates | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Format Slack Update | Code | Builds a short Slack message for ranking updates | Build Change Summary | Post Ranking Change | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Post Ranking Change | Slack | Sends ranking-change notification to Slack | Format Slack Update | Create Audit Log | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| No Changes | No Operation | Placeholder node for no-change branch | Any Changes | Slack No Changes | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Slack No Changes | Slack | Sends informational message when no ranking output is detected | No Changes | Create Audit Log | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Create Audit Log | Notion | Writes an audit entry to Notion | Post Ranking Change; Slack No Changes |  | ## Change Detection CompareDatasets identifies ranking differences. If changes, batch-updates Notion and posts before/after to Slack. |
| Sticky Note - How It Works | Sticky Note | Documentation note describing overall workflow behavior |  |  |  |
| Sticky Note - Context Collection | Sticky Note | Documentation note describing the context collection block |  |  |  |
| Sticky Note - Priority Matrix | Sticky Note | Documentation note describing matrix-building logic |  |  |  |
| Sticky Note - 3-Pass AI Scoring | Sticky Note | Documentation note describing AI scoring phases |  |  |  |
| Sticky Note - Change Detection | Sticky Note | Documentation note describing comparison and update behavior |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `PM Daily Smart Priority Reranker`.
   - Keep it inactive until credentials and databases are verified.

2. **Add the trigger**
   - Create a **Schedule Trigger** node named `Rerank Trigger`.
   - Configure a cron schedule:
     - `0 */2 8-18 * * 1-5`
   - This runs every 2 hours on weekdays between 08:00 and 18:00.

3. **Add settings retrieval**
   - Create a **Notion** node named `Read Settings`.
   - Resource: `Database Page`
   - Operation: `Get Many` / `Get All`
   - Database ID: `31e06bab-3ebe-813a-858d-dfec8df6b644`
   - Enable return-all behavior.
   - Connect `Rerank Trigger -> Read Settings`.

4. **Add runtime gate**
   - Create an **If** node named `Check Demo Mode`.
   - Condition:
     - Left value: `={{ $json.properties?.demo_mode?.checkbox }}`
     - Operator: boolean `equals`
     - Right value: `true`
   - Connect `Read Settings -> Check Demo Mode`.
   - Only wire the **true** output onward if you want the same behavior as the exported workflow.

5. **Create the priorities branch**
   - Add a **Notion** node named `Get Open Priorities`.
   - Resource: `Database Page`
   - Operation: `Get All`
   - Database ID: `31e06bab-3ebe-81fe-9d0c-e65a49a99ef3`
   - Connect from `Check Demo Mode` true branch.

6. **Sort priorities**
   - Add a **Sort** node named `Sort Priorities`.
   - Connect `Get Open Priorities -> Sort Priorities`.
   - The exported workflow does not specify sort fields, but for a reliable rebuild you should explicitly sort by your current ranking property or priority field.

7. **Limit priorities**
   - Add a **Limit** node named `Limit Top 20`.
   - Set max items to `20`.
   - Connect `Sort Priorities -> Limit Top 20`.

8. **Normalize priorities**
   - Add a **Code** node named `Normalize Priorities`.
   - Paste logic equivalent to:
     - collect all incoming items
     - output one item with:
       - `source = "priorities"`
       - `items = [{ id, title, currentRank, status, priority, due }]`
   - Use these mappings:
     - title from `Name.title[0].plain_text`
     - status from `Status.select.name`
     - priority from `Priority.number`
     - due from `Due.date.start`
   - Connect `Limit Top 20 -> Normalize Priorities`.

9. **Create the signals branch**
   - Add a **Notion** node named `Get Recent Signals`.
   - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
   - Connect from `Check Demo Mode` true branch.

10. **Filter signals**
    - Add a **Filter** node named `Filter High Signals`.
    - Condition:
      - `={{ $json.properties?.Impact?.select?.name }}`
      - operator: `is not empty`
    - Connect `Get Recent Signals -> Filter High Signals`.

11. **Normalize signals**
    - Add a **Code** node named `Normalize Signals`.
    - Output one item:
      - `source = "signals"`
      - `items = [{ id, title, impact, category }]`
    - Connect `Filter High Signals -> Normalize Signals`.

12. **Create the meetings branch**
    - Add a **Notion** node named `Get Meeting Decisions`.
    - Database ID: `31e06bab-3ebe-8108-b9d7-cde4bd8192b6`
    - Connect from `Check Demo Mode` true branch.

13. **Normalize meetings**
    - Add a **Code** node named `Normalize Meetings`.
    - Output:
      - `source = "meetings"`
      - `items = [{ id, title, decisions }]`
    - Concatenate Notion rich-text decision fragments into one string.
    - Connect `Get Meeting Decisions -> Normalize Meetings`.

14. **Create the emails branch**
    - Add a **Notion** node named `Get Urgent Emails`.
    - Database ID: `31e06bab-3ebe-81d6-8c12-f607c8ff3b3f`
    - Connect from `Check Demo Mode` true branch.

15. **Filter urgent emails**
    - Add a **Filter** node named `Filter Urgent Emails`.
    - Condition:
      - `={{ $json.properties?.Priority?.select?.name }}`
      - equals `Urgent`
    - Connect `Get Urgent Emails -> Filter Urgent Emails`.

16. **Normalize emails**
    - Add a **Code** node named `Normalize Emails`.
    - Output:
      - `source = "emails"`
      - `items = [{ id, subject, sender, urgency }]`
    - Connect `Filter Urgent Emails -> Normalize Emails`.

17. **Create the blockers branch**
    - Add a **Notion** node named `Get Blocked Actions`.
    - Database ID: `31e06bab-3ebe-8190-894f-f211aea15dc4`
    - Connect from `Check Demo Mode` true branch.

18. **Filter blocked actions**
    - Add a **Filter** node named `Filter Overdue Actions`.
    - Condition:
      - `={{ $json.properties?.Status?.select?.name }}`
      - equals `Blocked`
    - Connect `Get Blocked Actions -> Filter Overdue Actions`.

19. **Normalize blockers**
    - Add a **Code** node named `Normalize Blockers`.
    - Output:
      - `source = "blockers"`
      - `items = [{ id, title, status, due }]`
    - Connect `Filter Overdue Actions -> Normalize Blockers`.

20. **Merge all context branches**
    - Add a **Merge** node named `Merge Context`.
    - Set mode to `Combine`.
    - Connect into it:
      - `Normalize Priorities`
      - `Normalize Signals`
      - `Normalize Meetings`
      - `Normalize Emails`
      - `Normalize Blockers`

21. **Build the priority matrix**
    - Add a **Code** node named `Build Priority Matrix`.
    - Implement logic to:
      - read all combined inputs
      - reconstruct arrays by `source`
      - for each priority, find related signals, emails, and blockers using title-word matching
      - return one item containing:
        - `matrix`
        - `meetings`
        - `totalPriorities`
        - `totalSignals`
        - `totalBlockers`
   - Connect `Merge Context -> Build Priority Matrix`.

22. **Add deduplication**
   - Create a **Remove Duplicates** node named `Remove Duplicates`.
   - Connect `Build Priority Matrix -> Remove Duplicates`.
   - The export leaves this minimally configured; if rebuilding for production, specify the field(s) to deduplicate on.

23. **Add first OpenAI pass**
   - Create an **OpenAI Chat** node named `AI Pass 1 - Impact`.
   - Resource: `Chat`
   - Connect `Remove Duplicates -> AI Pass 1 - Impact`.
   - Configure OpenAI credentials.
   - Add a system/user prompt instructing the model to score impact for each priority and return strict JSON, ideally:
     - `{ "impactScores": [...] }`

24. **Parse impact output**
   - Add a **Code** node named `Parse Impact`.
   - Parse `message.content`, extract JSON with a regex, then map to:
     - `impactScores`
   - Connect `AI Pass 1 - Impact -> Parse Impact`.

25. **Store impact result**
   - Add a **Set** node named `Set Impact Scores`.
   - Add field:
     - `impact` as object
     - value: `={{ $json }}`
   - Connect `Parse Impact -> Set Impact Scores`.

26. **Add second OpenAI pass**
   - Create an **OpenAI Chat** node named `AI Pass 2 - Urgency`.
   - Connect `Set Impact Scores -> AI Pass 2 - Urgency`.
   - Configure prompt to calculate urgency adjustments and return JSON such as:
     - `{ "urgencyScores": [...] }`

27. **Parse urgency output**
   - Add a **Code** node named `Parse Urgency`.
   - Same parsing pattern as impact, but output:
     - `urgencyScores`
   - Connect `AI Pass 2 - Urgency -> Parse Urgency`.

28. **Store urgency result**
   - Add a **Set** node named `Set Urgency Scores`.
   - Add field:
     - `urgency` as object
     - value: `={{ $json }}`
   - Connect `Parse Urgency -> Set Urgency Scores`.

29. **Add final OpenAI pass**
   - Create an **OpenAI Chat** node named `AI Pass 3 - Final Rank`.
   - Connect `Set Urgency Scores -> AI Pass 3 - Final Rank`.
   - Prompt the model to output final rank ordering with rationale and change metadata, ideally:
     - `{ "ranking": [{ "id": "...", "rank": 1, "title": "...", "finalScore": 0, "change": "up|down|same", "rationale": "..." }] }`

30. **Parse final ranking**
   - Add a **Code** node named `Parse Ranking`.
   - Parse `message.content` and output:
     - `newRanking`
   - Connect `AI Pass 3 - Final Rank -> Parse Ranking`.

31. **Add comparison step**
   - Create a **Compare Datasets** node named `Compare Rankings`.
   - Merge by field:
     - `id` to `id`
   - Connect `Parse Ranking -> Compare Rankings`.
   - For a robust rebuild, also provide the baseline/current ranking as a second dataset input; the exported workflow does not show that connection clearly.

32. **Create routing condition**
   - Add an **If** node named `Any Changes`.
   - Condition:
     - Left value: `={{ $json.newRanking?.length > 0 }}`
     - string equals `true`
   - Connect `Compare Rankings -> Any Changes`.

33. **Build detailed change summary**
   - Add a **Code** node named `Build Change Summary`.
   - Generate:
     - `changeSummary`
     - `changedCount`
   - Include ranking lines with arrows based on `change`.
   - Connect from `Any Changes` true branch.

34. **Build Slack-friendly message**
   - Add a **Code** node named `Format Slack Update`.
   - Produce `slackText` summarizing the top 5 priorities.
   - Connect `Build Change Summary -> Format Slack Update`.

35. **Post Slack update**
   - Add a **Slack** node named `Post Ranking Change`.
   - Set text to:
     - `={{ $json.slackText }}`
   - Configure Slack credentials and destination channel.
   - Connect `Format Slack Update -> Post Ranking Change`.

36. **Prepare Notion updates**
   - Add a **Code** node named `Prepare Update Batch`.
   - Map `newRanking` into one item per page:
     - `pageId`
     - `rank`
     - `finalScore`
     - `rationale`
   - Connect from `Any Changes` true branch.

37. **Loop through updates**
   - Add a **Split In Batches** node named `Batch Updates`.
   - Connect `Prepare Update Batch -> Batch Updates`.

38. **Update Notion priority pages**
   - Add a **Notion** node named `Update Notion Priority`.
   - Operation: `Update Database Page`
   - Page ID:
     - `={{ $json.pageId }}`
   - Connect `Batch Updates -> Update Notion Priority`.
   - Then connect `Update Notion Priority -> Batch Updates` to continue looping.
   - Important: unlike the exported JSON, you should explicitly map the properties to update, for example:
     - Rank field
     - Final score field
     - Rationale field

39. **Add no-change placeholder**
   - Create a **No Operation** node named `No Changes`.
   - Connect from `Any Changes` false branch.

40. **Send no-change Slack message**
   - Add a **Slack** node named `Slack No Changes`.
   - Text:
     - `={{ ':white_check_mark: Priority reranker ran at ' + new Date().toISOString() + ' - no ranking changes detected.' }}`
   - Connect `No Changes -> Slack No Changes`.

41. **Create audit log**
   - Add a **Notion** node named `Create Audit Log`.
   - Configure it to create a page in database:
     - `31e06bab-3ebe-811b-af1f-f186b6dbdf3e`
   - Connect both:
     - `Post Ranking Change -> Create Audit Log`
     - `Slack No Changes -> Create Audit Log`
   - In a production rebuild, explicitly map audit properties such as:
     - timestamp
     - status
     - changed count
     - Slack message or summary

42. **Add credentials**
   - **Notion credential:** must have access to all six databases used:
     - settings
     - priorities
     - signals
     - meetings
     - emails
     - blockers
     - audit log
   - **OpenAI credential:** required for the three chat nodes.
   - **Slack credential:** required for both Slack posting nodes.

43. **Add sticky notes for maintainability**
   - Add sticky notes matching these sections if desired:
     - How it works
     - Context Collection
     - Priority Matrix
     - 3-Pass AI Scoring
     - Change Detection

44. **Test each branch independently**
   - Validate that each Notion database returns the expected schema:
     - `Name`
     - `Status`
     - `Priority`
     - `Due`
     - `Impact`
     - `Category`
     - `Decisions`
     - `Subject`
     - `From`
   - Confirm the property types match the code expectations.

45. **Harden the rebuild**
   - Recommended improvements beyond the exported structure:
     - explicitly filter priorities to open records
     - explicitly sort priorities by current rank
     - feed baseline ranking into `Compare Datasets`
     - ensure the Set nodes preserve prior context
     - force OpenAI strict JSON mode if available
     - define exact Notion properties to update in `Update Notion Priority`
     - define exact audit fields in `Create Audit Log`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow contains internal visual documentation via sticky notes explaining the five main phases: overall behavior, context collection, priority matrix construction, AI scoring, and change detection. | Internal workflow canvas notes |
| The workflow title provided by the user is “Rerank PM priorities every 2 hours using OpenAI, Notion, and Slack,” while the workflow object name is “PM Daily Smart Priority Reranker.” | Naming/context note |
| The exported workflow is inactive (`active: false`), so scheduling will not run until it is enabled. | Deployment note |
| Several operational details are implied but not fully configured in the export, especially OpenAI prompt contents, Notion update field mappings, and audit-log field mappings. | Implementation caution |
| There are no external links, branding URLs, or project-credit links present in the workflow content. | Resource note |