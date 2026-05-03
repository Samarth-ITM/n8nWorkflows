Monitor deal risk from HubSpot and Gmail with GPT-4.1 mini and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-deal-risk-from-hubspot-and-gmail-with-gpt-4-1-mini-and-slack-alerts-14271


# Monitor deal risk from HubSpot and Gmail with GPT-4.1 mini and Slack alerts

# 1. Workflow Overview

This workflow monitors HubSpot deals on a daily schedule, enriches the analysis with recent Gmail activity, uses GPT-4.1 mini to assess deal risk, writes structured insights back to HubSpot, and sends a Slack alert when a deal crosses a configured risk threshold.

Typical use cases include:

- Daily sales pipeline monitoring
- Early detection of stalled or at-risk deals
- Automated extraction of objections and next steps from communications
- CRM enrichment with AI-generated deal intelligence
- Escalation of high-risk opportunities to sales leadership

The workflow is organized into the following logical blocks:

## 1.1 Scheduled Start and Global Configuration
The workflow starts once per day and sets shared parameters such as the email lookback window and the threshold used to determine whether a deal is risky enough to trigger an alert.

## 1.2 Data Collection
The workflow retrieves all HubSpot deals and fetches Gmail messages received within a configurable number of days.

## 1.3 Context Consolidation
The separate streams of CRM and email data are merged into a single aggregated payload for downstream AI analysis.

## 1.4 AI Deal Risk Analysis
A LangChain Agent node calls OpenAI GPT-4.1 mini with a sales-operations risk-analysis prompt and enforces a structured JSON response using an output parser.

## 1.5 CRM Update Preparation and Writeback
The structured AI output is reshaped into CRM-ready fields and then written back to the HubSpot deal record.

## 1.6 Risk Threshold Evaluation and Slack Alerting
If the calculated risk score is greater than or equal to the configured threshold, the workflow sends a Slack alert to the target sales-management channel.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Start and Global Configuration

### Overview
This block starts the workflow every day and defines shared runtime parameters. These values are later referenced by expressions in Gmail filtering and risk evaluation.

### Nodes Involved
- Schedule Trigger
- Workflow Configuration

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger node that launches the workflow on a schedule.
- **Configuration choices:** Configured with an interval rule that triggers at hour `9`.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input, as it is the workflow trigger  
  - Outputs to `Workflow Configuration`
- **Version-specific requirements:** Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**
  - Time-zone mismatch if the n8n instance timezone is not what the business expects
  - Workflow may appear to run at the wrong local time if server timezone differs from team timezone
- **Sub-workflow reference:** None

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; creates reusable configuration fields for downstream nodes.
- **Configuration choices:** Adds three fields while keeping incoming fields:
  - `dealStageFilter` = `"negotiation,proposal"`
  - `emailLookbackDays` = `7`
  - `riskScoreThreshold` = `70`
- **Key expressions or variables used:** These values are later referenced as:
  - `$('Workflow Configuration').first().json.emailLookbackDays`
  - `$('Workflow Configuration').first().json.riskScoreThreshold`
- **Input and output connections:**
  - Input from `Schedule Trigger`
  - Outputs to both `Get CRM Deals` and `Get Recent Emails`
- **Version-specific requirements:** Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**
  - The `dealStageFilter` value is currently not used by any later node, so users may incorrectly assume deal filtering is active
  - If field names are changed here, downstream expressions will break
- **Sub-workflow reference:** None

---

## 2.2 Data Collection

### Overview
This block gathers source data from HubSpot and Gmail. It is intended to provide the AI with current CRM records plus recent customer communication context.

### Nodes Involved
- Get CRM Deals
- Get Recent Emails

### Node Details

#### Get CRM Deals
- **Type and technical role:** `n8n-nodes-base.hubspot`; retrieves HubSpot deal records.
- **Configuration choices:**
  - Resource: `deal`
  - Operation: `getAll`
  - `returnAll`: enabled
  - No filters are actually configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Output to `Combine Deal Context`
- **Version-specific requirements:** Uses `typeVersion 2.2`; requires valid HubSpot credentials.
- **Edge cases or potential failure types:**
  - HubSpot authentication failure or expired credentials
  - Large deal volume may increase execution time and memory usage
  - Since no stage filter is applied, all accessible deals are returned, not just negotiation/proposal deals
  - API rate limiting if account volume is high
- **Sub-workflow reference:** None

#### Get Recent Emails
- **Type and technical role:** `n8n-nodes-base.gmail`; retrieves recent Gmail messages.
- **Configuration choices:**
  - Operation: `getAll`
  - Filter `receivedAfter` is dynamically computed using the lookback period from `Workflow Configuration`
- **Key expressions or variables used:**
  - `={{ $now.minus({ days: $('Workflow Configuration').first().json.emailLookbackDays }).toISO() }}`
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Output to `Combine Deal Context`
- **Version-specific requirements:** Uses `typeVersion 2.2`; requires Gmail credentials with permission to read messages.
- **Edge cases or potential failure types:**
  - OAuth token expiration or missing Gmail scopes
  - If the mailbox is large, `getAll` may return many messages and affect performance
  - The emails are not filtered by deal, contact, thread, or domain, so irrelevant email content may be included
  - Expression failure if the configuration node name changes
- **Sub-workflow reference:** None

---

## 2.3 Context Consolidation

### Overview
This block aggregates the deal and email streams into a single combined payload. The result is passed as one JSON object to the AI analysis node.

### Nodes Involved
- Combine Deal Context

### Node Details

#### Combine Deal Context
- **Type and technical role:** `n8n-nodes-base.aggregate`; aggregates incoming item data into a single structure.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData`
  - No special options enabled
- **Key expressions or variables used:** None directly in the node configuration.
- **Input and output connections:**
  - Inputs from `Get CRM Deals` and `Get Recent Emails`
  - Output to `AI Deal Intelligence Analyzer`
- **Version-specific requirements:** Uses `typeVersion 1`.
- **Edge cases or potential failure types:**
  - The aggregation combines all incoming deal and email items globally rather than pairing emails to specific deals
  - This means the AI receives a broad mixed context instead of one record per deal
  - If the intent was per-deal analysis, the current design does not implement that
  - Large input sets may produce oversized prompts, leading to token limits or slower model responses
- **Sub-workflow reference:** None

---

## 2.4 AI Deal Risk Analysis

### Overview
This block sends the aggregated CRM and email context to an AI agent instructed to identify objections, next steps, risk signals, and a risk score. A structured output parser forces the AI response into a JSON schema.

### Nodes Involved
- AI Deal Intelligence Analyzer
- OpenAI Chat Model
- Deal Intelligence Schema

### Node Details

#### AI Deal Intelligence Analyzer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates an LLM call with a fixed prompt and structured-output enforcement.
- **Configuration choices:**
  - Prompt type: defined manually
  - Input text:
    - `Analyze the following CRM deal data and recent email communications: {{ $json }}`
  - System message instructs the model to:
    1. Extract objections
    2. Identify next steps
    3. Detect risk signals
    4. Calculate a 0–100 risk score
    5. Provide a brief risk summary
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{ $json }}` to inject the aggregated dataset
- **Input and output connections:**
  - Main input from `Combine Deal Context`
  - AI language model input from `OpenAI Chat Model`
  - AI output parser input from `Deal Intelligence Schema`
  - Main output to `Prepare CRM Update`
- **Version-specific requirements:** Uses `typeVersion 3`; requires compatible LangChain nodes installed in the n8n environment.
- **Edge cases or potential failure types:**
  - Invalid or oversized prompt due to too much aggregated data
  - Model may hallucinate a `dealId` if the context is ambiguous
  - Structured parsing may fail if the model returns invalid JSON
  - If the aggregated input does not clearly associate emails to deals, the risk score may be unreliable
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides the LLM used by the agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No additional options configured
  - No built-in tools enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connected to `AI Deal Intelligence Analyzer` through the `ai_languageModel` port
- **Version-specific requirements:** Uses `typeVersion 1.3`; requires OpenAI credentials and model access for `gpt-4.1-mini`.
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model access not enabled on the OpenAI account
  - Rate limits or request timeouts
  - Region or account restrictions
- **Sub-workflow reference:** None

#### Deal Intelligence Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates and structures the AI response against a JSON schema.
- **Configuration choices:** Manual schema with the following fields:
  - `objections`: array of strings
  - `nextSteps`: array of strings
  - `riskSignals`: array of strings
  - `riskScore`: number
  - `riskSummary`: string
  - `dealId`: string
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connected to `AI Deal Intelligence Analyzer` through the `ai_outputParser` port
- **Version-specific requirements:** Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**
  - Parsing failure if the model output does not match the schema
  - Schema does not mark fields as required, so incomplete outputs may still pass depending on parser behavior
  - `dealId` may be empty or unrelated if the source data is not clearly structured
- **Sub-workflow reference:** None

---

## 2.5 CRM Update Preparation and Writeback

### Overview
This block converts the AI response into a flatter, CRM-ready structure and updates custom HubSpot deal properties with the generated insights.

### Nodes Involved
- Prepare CRM Update
- Update CRM Deal

### Node Details

#### Prepare CRM Update
- **Type and technical role:** `n8n-nodes-base.set`; formats structured AI output into simple scalar fields.
- **Configuration choices:** Creates:
  - `dealId` from AI output
  - `objections` as semicolon-separated string
  - `nextSteps` as semicolon-separated string
  - `riskScore`
  - `riskSummary`
  - Keeps all other incoming fields
- **Key expressions or variables used:**
  - `={{ $json.dealId }}`
  - `={{ $json.objections.join('; ') }}`
  - `={{ $json.nextSteps.join('; ') }}`
  - `={{ $json.riskScore }}`
  - `={{ $json.riskSummary }}`
- **Input and output connections:**
  - Input from `AI Deal Intelligence Analyzer`
  - Output to `Update CRM Deal`
- **Version-specific requirements:** Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**
  - `.join('; ')` fails if `objections` or `nextSteps` are missing or not arrays
  - Empty `dealId` prevents correct HubSpot update
  - Preserving all fields may carry more data than needed
- **Sub-workflow reference:** None

#### Update CRM Deal
- **Type and technical role:** `n8n-nodes-base.hubspot`; updates a specific HubSpot deal.
- **Configuration choices:**
  - Resource: `deal`
  - Operation: `update`
  - Deal ID comes from `={{ $json.dealId }}`
  - Writes custom properties:
    - `objections`
    - `next_steps`
    - `risk_score`
    - `risk_summary`
- **Key expressions or variables used:**
  - Deal ID: `={{ $json.dealId }}`
  - Property values:
    - `={{ $json.objections }}`
    - `={{ $json.next_steps }}`
    - `={{ $json.risk_score }}`
    - `={{ $json.risk_summary }}`
- **Input and output connections:**
  - Input from `Prepare CRM Update`
  - Output to `Check Deal Risk`
- **Version-specific requirements:** Uses `typeVersion 2.2`; requires HubSpot credentials and existing custom properties in HubSpot.
- **Edge cases or potential failure types:**
  - There is a field-name mismatch here:
    - `Prepare CRM Update` creates `nextSteps`, `riskScore`, `riskSummary`
    - `Update CRM Deal` references `next_steps`, `risk_score`, `risk_summary`
  - As currently configured, three HubSpot properties may receive empty values unless upstream data already includes those snake_case fields
  - HubSpot update fails if custom properties do not exist
  - Invalid deal ID or insufficient permissions will cause API errors
- **Sub-workflow reference:** None

---

## 2.6 Risk Threshold Evaluation and Slack Alerting

### Overview
This block checks whether the AI-generated risk score meets or exceeds the configured threshold. If true, it sends a formatted Slack alert to a designated channel.

### Nodes Involved
- Check Deal Risk
- Alert Sales Manager

### Node Details

#### Check Deal Risk
- **Type and technical role:** `n8n-nodes-base.if`; routes execution based on a numeric threshold condition.
- **Configuration choices:**
  - Condition: `riskScore >= riskScoreThreshold`
  - Comparison type: number, greater than or equal
- **Key expressions or variables used:**
  - Left value: `={{ $('Prepare CRM Update').item.json.riskScore }}`
  - Right value: `={{ $('Workflow Configuration').first().json.riskScoreThreshold }}`
- **Input and output connections:**
  - Input from `Update CRM Deal`
  - True output to `Alert Sales Manager`
  - False branch not connected
- **Version-specific requirements:** Uses `typeVersion 2.3`.
- **Edge cases or potential failure types:**
  - If `riskScore` is null or non-numeric, comparison may behave unexpectedly
  - False path is intentionally unused, so low-risk deals simply stop here
  - Uses cross-node reference instead of current item field, which is valid but slightly more fragile during refactoring
- **Sub-workflow reference:** None

#### Alert Sales Manager
- **Type and technical role:** `n8n-nodes-base.slack`; sends a Slack message to a channel.
- **Configuration choices:**
  - Sends plain formatted message text
  - Channel mode: fixed channel ID
  - Channel ID is still a placeholder
- **Key expressions or variables used:**
  - Message includes:
    - `{{ $json.dealId }}`
    - `{{ $json.riskScore }}`
    - `{{ $json.riskSummary }}`
    - `{{ $json.objections }}`
    - `{{ $json.nextSteps }}`
- **Input and output connections:**
  - Input from true branch of `Check Deal Risk`
  - No downstream output
- **Version-specific requirements:** Uses `typeVersion 2.4`; requires Slack credentials with permission to post in the target channel.
- **Edge cases or potential failure types:**
  - Channel ID placeholder must be replaced before production use
  - Slack auth or bot membership issues can prevent message delivery
  - Formatting may be suboptimal if objections or next steps are very long
  - The message references `nextSteps`, which exists from `Prepare CRM Update`, so this part is consistent
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow every day at the configured hour |  | Workflow Configuration | ## Trigger<br>Runs daily to start deal monitoring workflow. |
| Workflow Configuration | n8n-nodes-base.set | Stores shared runtime settings such as lookback days and risk threshold | Schedule Trigger | Get CRM Deals; Get Recent Emails | ## Configuration<br>Defines deal filters, email lookback window, and risk threshold. |
| Get CRM Deals | n8n-nodes-base.hubspot | Retrieves HubSpot deal records | Workflow Configuration | Combine Deal Context | ## Data Collection<br>Fetches CRM deals and recent email activity for context. |
| Get Recent Emails | n8n-nodes-base.gmail | Retrieves recent Gmail messages within the configured time window | Workflow Configuration | Combine Deal Context | ## Data Collection<br>Fetches CRM deals and recent email activity for context. |
| Combine Deal Context | n8n-nodes-base.aggregate | Aggregates all incoming deal and email data into one payload | Get CRM Deals; Get Recent Emails | AI Deal Intelligence Analyzer | ## Context Merge<br>Combines deal data and emails into a single dataset for analysis. |
| AI Deal Intelligence Analyzer | @n8n/n8n-nodes-langchain.agent | Sends combined context to the AI model for structured risk analysis | Combine Deal Context | Prepare CRM Update | ## AI Analysis<br>AI detects objections, next steps, and calculates deal risk score. |
| Deal Intelligence Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output from the AI agent |  | AI Deal Intelligence Analyzer | ## AI Analysis<br>AI detects objections, next steps, and calculates deal risk score. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the GPT-4.1 mini model used by the AI agent |  | AI Deal Intelligence Analyzer | ## AI Analysis<br>AI detects objections, next steps, and calculates deal risk score. |
| Prepare CRM Update | n8n-nodes-base.set | Reshapes AI output into fields intended for HubSpot update | AI Deal Intelligence Analyzer | Update CRM Deal | ## Data Preparation and update<br>Formats AI output for updating CRM fields and Writes risk insights, objections, and next steps back to CRM. |
| Update CRM Deal | n8n-nodes-base.hubspot | Updates HubSpot deal properties with AI-generated insights | Prepare CRM Update | Check Deal Risk | ## Data Preparation and update<br>Formats AI output for updating CRM fields and Writes risk insights, objections, and next steps back to CRM. |
| Check Deal Risk | n8n-nodes-base.if | Compares risk score against the configured alert threshold | Update CRM Deal | Alert Sales Manager | ## Risk Check and Alerting<br>Compares deal risk score against defined threshold and Sends Slack alert for high-risk deals needing attention. |
| Alert Sales Manager | n8n-nodes-base.slack | Sends Slack alert for high-risk deals | Check Deal Risk |  | ## Risk Check and Alerting<br>Compares deal risk score against defined threshold and Sends Slack alert for high-risk deals needing attention. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for the trigger area |  |  | ## Trigger<br>Runs daily to start deal monitoring workflow. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for configuration |  |  | ## Configuration<br>Defines deal filters, email lookback window, and risk threshold. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for context merge |  |  | ## Context Merge<br>Combines deal data and emails into a single dataset for analysis. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for AI analysis |  |  | ## AI Analysis<br>AI detects objections, next steps, and calculates deal risk score. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note for CRM writeback |  |  | ## Data Preparation and update<br>Formats AI output for updating CRM fields and Writes risk insights, objections, and next steps back to CRM. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note for risk check and alerting |  |  | ## Risk Check and Alerting<br>Compares deal risk score against defined threshold and Sends Slack alert for high-risk deals needing attention. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Global explanatory note with setup guidance |  |  | ## How it works<br>This workflow monitors your CRM deals and flags risky ones automatically.<br><br>It runs on a schedule and pulls active deals from your CRM along with recent email conversations. The data is combined and analyzed using AI to understand deal progress, objections, and engagement.<br><br>The AI identifies key issues, suggests next steps, and assigns a risk score based on signals like delays, lack of response, or pricing concerns.<br><br>The results are saved back into the CRM as structured fields so your team has clear visibility.<br><br>If a deal crosses a defined risk threshold, the workflow sends a Slack alert to notify the sales manager for immediate action.<br><br>## Setup steps<br>1. Connect HubSpot credentials (for deals)<br>2. Connect Gmail (to fetch recent emails)<br>3. Connect OpenAI (for analysis)<br>4. Connect Slack (for alerts)<br>5. Set risk threshold and filters in config node<br>6. Test with sample deals<br>7. Activate the workflow |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note for data collection |  |  | ## Data Collection<br>Fetches CRM deals and recent email activity for context. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Monitor deal risk from HubSpot and Gmail with GPT-4.1 mini and Slack alerts`

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run once daily at hour `9`
   - This becomes the workflow entry point

3. **Add a Set node named `Workflow Configuration`**
   - Node type: `Set`
   - Enable keeping other input fields
   - Create these fields:
     - `dealStageFilter` as string: `negotiation,proposal`
     - `emailLookbackDays` as number: `7`
     - `riskScoreThreshold` as number: `70`
   - Connect `Schedule Trigger -> Workflow Configuration`

4. **Add a HubSpot node named `Get CRM Deals`**
   - Node type: `HubSpot`
   - Credentials: connect a HubSpot account with access to deals
   - Resource: `Deal`
   - Operation: `Get Many` / `Get All` depending on your n8n UI wording
   - Enable `Return All`
   - Leave filters empty to match the original workflow
   - Connect `Workflow Configuration -> Get CRM Deals`

5. **Add a Gmail node named `Get Recent Emails`**
   - Node type: `Gmail`
   - Credentials: connect Gmail OAuth2 with read access to messages
   - Operation: `Get Many` / `Get All`
   - In filters, set `Received After` to this expression:  
     `{{ $now.minus({ days: $('Workflow Configuration').first().json.emailLookbackDays }).toISO() }}`
   - Connect `Workflow Configuration -> Get Recent Emails`

6. **Add an Aggregate node named `Combine Deal Context`**
   - Node type: `Aggregate`
   - Set aggregate mode to `Aggregate All Item Data`
   - Connect:
     - `Get CRM Deals -> Combine Deal Context`
     - `Get Recent Emails -> Combine Deal Context`

7. **Add an AI Agent node named `AI Deal Intelligence Analyzer`**
   - Node type: `AI Agent` / LangChain Agent
   - Set prompt mode to manually defined prompt
   - In the main text/input field, use:  
     `Analyze the following CRM deal data and recent email communications: {{ $json }}`
   - In system message, paste:

     ```text
     You are an AI sales operations analyst specializing in deal intelligence and risk assessment.

     Your task is to:
     1. Extract key objections mentioned in emails or call notes
     2. Identify clear next steps and action items
     3. Detect risk signals such as:
        - Delayed responses or ghosting
        - Budget concerns or pricing objections
        - Competitor mentions
        - Stakeholder changes or lack of engagement
        - Timeline pushbacks
     4. Calculate a risk score from 0-100 (0=no risk, 100=deal likely to be lost)
     5. Provide a brief risk summary

     Return your analysis in the structured JSON format defined by the output parser.
     ```

   - Enable structured output / output parser support if your UI requires it
   - Connect `Combine Deal Context -> AI Deal Intelligence Analyzer`

8. **Add an OpenAI Chat Model node named `OpenAI Chat Model`**
   - Node type: `OpenAI Chat Model`
   - Credentials: connect OpenAI API credentials
   - Select model: `gpt-4.1-mini`
   - Leave optional settings at defaults unless your environment requires temperature or timeout adjustments
   - Connect the model node to the AI Agent through the `Language Model` port

9. **Add a Structured Output Parser node named `Deal Intelligence Schema`**
   - Node type: `Structured Output Parser`
   - Choose manual JSON schema
   - Paste this schema:

     ```json
     {
       "type": "object",
       "properties": {
         "objections": {
           "type": "array",
           "items": {
             "type": "string"
           }
         },
         "nextSteps": {
           "type": "array",
           "items": {
             "type": "string"
           }
         },
         "riskSignals": {
           "type": "array",
           "items": {
             "type": "string"
           }
         },
         "riskScore": {
           "type": "number"
         },
         "riskSummary": {
           "type": "string"
         },
         "dealId": {
           "type": "string"
         }
       }
     }
     ```
   - Connect the parser node to the AI Agent through the `Output Parser` port

10. **Add a Set node named `Prepare CRM Update`**
    - Node type: `Set`
    - Keep other fields enabled
    - Create these fields:
      - `dealId` as expression: `{{ $json.dealId }}`
      - `objections` as expression: `{{ $json.objections.join('; ') }}`
      - `nextSteps` as expression: `{{ $json.nextSteps.join('; ') }}`
      - `riskScore` as expression: `{{ $json.riskScore }}`
      - `riskSummary` as expression: `{{ $json.riskSummary }}`
    - Connect `AI Deal Intelligence Analyzer -> Prepare CRM Update`

11. **Add a HubSpot node named `Update CRM Deal`**
    - Node type: `HubSpot`
    - Credentials: same HubSpot connection as before
    - Resource: `Deal`
    - Operation: `Update`
    - Deal ID: `{{ $json.dealId }}`
    - Add custom property mappings for:
      - Property `objections` = `{{ $json.objections }}`
      - Property `next_steps` = `{{ $json.next_steps }}`
      - Property `risk_score` = `{{ $json.risk_score }}`
      - Property `risk_summary` = `{{ $json.risk_summary }}`
    - Connect `Prepare CRM Update -> Update CRM Deal`

12. **Important correction recommended during rebuild**
    - The original workflow contains a naming mismatch.
    - To make the workflow actually work as intended, either:
      - change `Prepare CRM Update` to create `next_steps`, `risk_score`, and `risk_summary`, or
      - change `Update CRM Deal` to reference `nextSteps`, `riskScore`, and `riskSummary`
    - The cleaner option is usually to update the HubSpot node to:
      - `next_steps` = `{{ $json.nextSteps }}`
      - `risk_score` = `{{ $json.riskScore }}`
      - `risk_summary` = `{{ $json.riskSummary }}`

13. **Ensure HubSpot custom properties exist**
    - In HubSpot, create or verify custom deal properties:
      - `objections`
      - `next_steps`
      - `risk_score`
      - `risk_summary`
    - Property types should be compatible with the values being written:
      - text for `objections`, `next_steps`, `risk_summary`
      - number for `risk_score`

14. **Add an If node named `Check Deal Risk`**
    - Node type: `If`
    - Configure one numeric condition:
      - Left value: `{{ $('Prepare CRM Update').item.json.riskScore }}`
      - Operation: `Greater Than or Equal`
      - Right value: `{{ $('Workflow Configuration').first().json.riskScoreThreshold }}`
    - Connect `Update CRM Deal -> Check Deal Risk`

15. **Add a Slack node named `Alert Sales Manager`**
    - Node type: `Slack`
    - Credentials: connect Slack app credentials authorized to post messages
    - Action: send message to channel
    - Choose channel by fixed channel ID
    - Replace the placeholder with the real sales-alert channel ID
    - Message text:

      ```text
      🚨 *High-Risk Deal Alert*

      *Deal ID:* {{ $json.dealId }}
      *Risk Score:* {{ $json.riskScore }}/100
      *Risk Summary:* {{ $json.riskSummary }}

      *Key Objections:*
      {{ $json.objections }}

      *Next Steps:*
      {{ $json.nextSteps }}

      Please review this deal immediately.
      ```

    - Connect the **true** output of `Check Deal Risk -> Alert Sales Manager`

16. **Optionally add sticky notes** matching the original layout
    - Trigger
    - Configuration
    - Data Collection
    - Context Merge
    - AI Analysis
    - Data Preparation and update
    - Risk Check and Alerting
    - Global “How it works” note

17. **Test each credential separately**
    - HubSpot: confirm deal listing and deal update permissions
    - Gmail: confirm message retrieval works and scopes are sufficient
    - OpenAI: confirm access to `gpt-4.1-mini`
    - Slack: confirm the bot/app can post to the selected channel

18. **Run the workflow manually**
    - Confirm:
      - HubSpot deals are returned
      - Gmail emails are returned
      - Aggregate output is not too large
      - AI output matches the schema
      - `dealId` is populated
      - HubSpot properties update correctly
      - Slack message is sent only when `riskScore >= threshold`

19. **Validate the design logic before activation**
    - The original flow aggregates all deals and all emails into one prompt
    - If you want one risk result per deal, you should redesign the middle section to:
      - iterate through deals one by one
      - search only emails related to that specific deal/contact/company
      - call the AI once per deal
    - Without that redesign, the workflow may produce only one generalized analysis instead of deal-specific analysis

20. **Activate the workflow** once test executions confirm correct behavior.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow monitors CRM deals and flags risky ones automatically. | Global workflow concept |
| It runs on a schedule and pulls active deals from the CRM along with recent email conversations. | Process summary |
| The data is combined and analyzed using AI to understand deal progress, objections, and engagement. | Process summary |
| The AI identifies key issues, suggests next steps, and assigns a risk score based on signals like delays, lack of response, or pricing concerns. | Process summary |
| The results are saved back into the CRM as structured fields so the team has clear visibility. | CRM enrichment purpose |
| If a deal crosses a defined risk threshold, the workflow sends a Slack alert to notify the sales manager for immediate action. | Alerting purpose |
| Setup steps: Connect HubSpot credentials, Gmail, OpenAI, Slack, set threshold and filters in configuration, test with sample deals, then activate. | Operational setup guidance |

## Additional implementation notes
- The workflow has **one entry point**: `Schedule Trigger`.
- The workflow has **no sub-workflow nodes** and does not invoke any child workflows.
- The `dealStageFilter` configuration is currently **unused**.
- The current design does **not actually correlate emails to specific deals**.
- The HubSpot update node contains a **field naming inconsistency** that should be corrected before production use.