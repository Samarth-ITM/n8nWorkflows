BeyondPresence Sales Intelligence → Real-Time Lead Scoring

https://n8nworkflows.xyz/workflows/beyondpresence-sales-intelligence---real-time-lead-scoring-4454


# BeyondPresence Sales Intelligence → Real-Time Lead Scoring

---

### 1. Workflow Overview

This workflow, titled **BeyondPresence Sales Opportunity Scoring - Real-Time Lead Intelligence**, is designed to monitor real-time user messages from BeyondPresence calls, score these leads based on predefined buying and negative signals, detect competitor mentions, estimate potential deal sizes, and send actionable alerts to Slack channels. It serves sales teams aiming to prioritize leads, identify urgent opportunities, and gain competitive intelligence during ongoing conversations.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and Filtering**: Captures and acknowledges incoming BeyondPresence webhook events, filtering user messages and call-ended events.
- **1.2 Lead Scoring Engine**: Processes individual user messages to compute sales opportunity scores, detect signals, competitor mentions, and estimate deal sizes.
- **1.3 Alert Routing and Notifications**: Routes scored results into specific Slack channels for hot leads, competitor alerts, and qualified leads.
- **1.4 Call Summary Processing**: Aggregates entire call conversation data at call end to produce summary reports and qualification levels.
- **1.5 Configuration and Documentation**: Provides sticky notes containing workflow overview, scoring rules, alert descriptions, and setup instructions for maintainers.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Filtering

**Overview:**  
This block receives incoming HTTP POST requests from BeyondPresence via a webhook, acknowledges receipt, and filters events, separating user messages from call-ended notifications.

**Nodes Involved:**  
- BeyondPresence Real-Time Webhook  
- Acknowledge Receipt  
- Filter User Messages  

**Node Details:**

- **BeyondPresence Real-Time Webhook**  
  - *Type:* Webhook (HTTP POST listener)  
  - *Role:* Entry point capturing real-time BeyondPresence call events.  
  - *Configuration:* Listens on path `beyondpresence-sales-intelligence` with POST method; returns response via connected node.  
  - *Input/Output:* Receives JSON payload with call and message details; outputs raw JSON to next node.  
  - *Failure Modes:* Network issues, malformed request bodies, missing required fields could cause failures.  
  - *Version:* TypeVersion 2.

- **Acknowledge Receipt**  
  - *Type:* Respond to Webhook  
  - *Role:* Immediately returns HTTP 200 with a JSON success message and timestamp to confirm event receipt.  
  - *Configuration:* Static JSON response confirming processing status with timestamp expression `{{ $now.toISO() }}`.  
  - *Input/Output:* Input from webhook node; outputs nothing further downstream except confirmation response.  
  - *Failure Modes:* Unlikely to fail unless expression `{{ $now.toISO() }}` is malformed, which is standard and safe.  
  - *Version:* TypeVersion 1.2.

- **Filter User Messages**  
  - *Type:* Switch  
  - *Role:* Branches incoming events into user messages (`user_message`) or call ended events (`call_ended`), based on event_type and sender fields.  
  - *Configuration:*  
    - Output `user_message`: event_type equals `"message"` AND message.sender equals `"user"`.  
    - Output `call_ended`: event_type equals `"call_ended"`.  
    - Fallback output is none (drops unmatched).  
  - *Input/Output:* Receives webhook JSON; outputs filtered streams for further processing.  
  - *Failure Modes:* If event_type or message.sender fields are missing or unexpected, event may be dropped or misrouted.  
  - *Version:* TypeVersion 3.2.

---

#### 2.2 Lead Scoring Engine

**Overview:**  
Processes individual user messages to score sales opportunity levels based on detecting positive buying signals, negative objections, competitor mentions, and estimated deal size. Outputs detailed scoring data, alerts, and recommended sales actions.

**Nodes Involved:**  
- Score Sales Opportunity  

**Node Details:**

- **Score Sales Opportunity**  
  - *Type:* Code  
  - *Role:* Core scoring logic implemented in JavaScript; analyzes message content to assign numeric scores and flags.  
  - *Configuration Highlights:*  
    - Defines positive buying signals keywords with associated point values (e.g., "purchase" +30, "demo" +25).  
    - Defines negative signals with negative points (e.g., "too expensive" -25, "not interested" -30).  
    - Detects competitor mentions from a customizable list.  
    - Estimates deal size category and multiplies estimated deal value accordingly.  
    - Computes lead temperature (Cold, Cool, Warm, Hot) based on score thresholds.  
    - Generates actionable recommendations based on score and detected signals.  
  - *Key Expressions:* Uses input JSON path `$input.first().json.body` for message data; uses regex to extract numeric deal values; constructs arrays for detected signals and alerts.  
  - *Input/Output:* Receives filtered user message JSON; outputs enriched JSON with scoring results, alerts, competitor info, deal estimates, and recommendations.  
  - *Failure Modes:*  
    - Missing or malformed input JSON structure will throw error and produce error output.  
    - Regex-based number extraction may fail or misinterpret numbers with unusual formats.  
    - Competitor keywords require manual update to maintain accuracy.  
  - *Version:* TypeVersion 2.

---

#### 2.3 Alert Routing and Notifications

**Overview:**  
Routes scored messages based on alert urgency and lead qualification, sending formatted notifications to Slack channels dedicated to hot leads, competitor intelligence, and qualified leads.

**Nodes Involved:**  
- Route Alerts  
- Slack: Hot Lead Alert  
- Slack: Competitor Alert  
- Slack: Qualified Lead  

**Node Details:**

- **Route Alerts**  
  - *Type:* Switch  
  - *Role:* Routes scored messages into specific alert categories based on JSON fields:  
    - `hot_lead`: alertUrgency equals `"high"`  
    - `competitor_alert`: hasCompetitorMention equals `true`  
    - `qualified_lead`: opportunityScore >= 40  
  - *Configuration:*  
    - Outputs renamed for clarity; fallback output is none (drops unmatched).  
  - *Input/Output:* Receives scored JSON; outputs to Slack alert nodes accordingly.  
  - *Failure Modes:* Incorrect or missing alertUrgency or hasCompetitorMention fields could misroute alerts.  
  - *Version:* TypeVersion 3.2.

- **Slack: Hot Lead Alert**  
  - *Type:* Slack node  
  - *Role:* Sends rich markdown alert message for hot leads to `#sales-hot-leads` Slack channel.  
  - *Configuration:*  
    - Text includes lead name, score, temperature, call ID, message, detected signals, recommended actions, and estimated deal value if available.  
    - Channel configured by name `#sales-hot-leads`.  
    - Uses OAuth2 Slack credentials for authentication.  
  - *Input/Output:* Input from route alerts; no output further.  
  - *Failure Modes:* Slack API rate limits, OAuth token expiration, or channel name mismatches could cause failures.  
  - *Version:* TypeVersion 2.2.

- **Slack: Competitor Alert**  
  - *Type:* Slack node  
  - *Role:* Sends alerts highlighting competitor mentions to `#sales-competitors` Slack channel.  
  - *Configuration:*  
    - Includes lead info, competitor names mentioned, message context, current lead score, and suggested responses.  
    - Channel configured by name `#sales-competitors`.  
    - Uses OAuth2 Slack credentials.  
  - *Failure Modes:* Similar to Hot Lead alert node.  
  - *Version:* TypeVersion 2.2.

- **Slack: Qualified Lead**  
  - *Type:* Slack node  
  - *Role:* Sends updates for qualified leads (score >= 40) to `#sales-qualified` Slack channel.  
  - *Configuration:*  
    - Presents lead info, latest message, positive buying signals (filters out negative signals), and next steps recommendations.  
    - Channel set as `#sales-qualified`.  
    - Uses OAuth2 Slack credentials.  
  - *Failure Modes:* Same as above Slack nodes.  
  - *Version:* TypeVersion 2.2.

---

#### 2.4 Call Summary Processing

**Overview:**  
Upon call completion, aggregates all user messages in the call to compute a final qualification score, summarize detected signals and competitors, and sends a call summary report to Slack.

**Nodes Involved:**  
- Process Call Summary  
- Slack: Call Summary  

**Node Details:**

- **Process Call Summary**  
  - *Type:* Code  
  - *Role:* Processes entire call data to produce aggregate scoring and qualification level.  
  - *Configuration Highlights:*  
    - Iterates through all user messages, accumulating scores based on buying and negative keywords.  
    - Tracks competitor mentions and maximum deal value (not explicitly multiplied here).  
    - Calculates average message score and assigns qualification level with emoji labels.  
    - Generates recommendations and next best action based on final score.  
  - *Input/Output:* Input receives call data with messages and metadata; outputs JSON summary with aggregated metrics and recommendations.  
  - *Failure Modes:* Missing messages array or unexpected call data structure could cause errors.  
  - *Version:* TypeVersion 2.

- **Slack: Call Summary**  
  - *Type:* Slack node  
  - *Role:* Posts a formatted call summary report to `#sales-summaries` Slack channel.  
  - *Configuration:*  
    - Includes lead, call duration, message counts, final qualification and score, signals detected, competitor mentions, next best action, and recommendations.  
    - Channel set as `#sales-summaries`.  
    - Uses OAuth2 Slack credentials.  
  - *Failure Modes:* Slack API or credential issues as previously noted.  
  - *Version:* TypeVersion 2.2.

---

#### 2.5 Configuration and Documentation

**Overview:**  
Sticky notes provide workflow context, scoring rules, alert descriptions, and setup instructions.

**Nodes Involved:**  
- Workflow Overview (sticky note)  
- Scoring Rules (sticky note)  
- Alert System (sticky note)  
- Setup Instructions (sticky note)  

**Node Details:**

- **Workflow Overview**  
  - *Type:* Sticky Note  
  - *Content:* Summarizes workflow goals: real-time monitoring, lead scoring, competitor detection, deal size estimation, Slack alerts.  
  - *Setup notes:* Slack OAuth2 credential configuration, Slack channel names, webhook URL.  

- **Scoring Rules**  
  - *Type:* Sticky Note  
  - *Content:* Details positive and negative buying signals with associated scoring points.  
  - *Purpose:* Reference for sales logic applied in code nodes.

- **Alert System**  
  - *Type:* Sticky Note  
  - *Content:* Describes Slack alerting logic, conditions for hot leads, competitor alerts, lead threading, and score tracking.  

- **Setup Instructions**  
  - *Type:* Sticky Note  
  - *Content:* Stepwise configuration instructions for Slack OAuth2 credentials, channel updates, competitor customization, and webhook testing.

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                            | Input Node(s)              | Output Node(s)                             | Sticky Note                                                                                 |
|-------------------------------|----------------------|------------------------------------------|----------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------|
| Workflow Overview              | Sticky Note          | Provides workflow purpose and setup info | -                          | -                                          | ## 🎯 Sales Opportunity Scoring System ... (details on goals and setup)                      |
| Scoring Rules                 | Sticky Note          | Explains scoring logic                    | -                          | -                                          | ## 📊 Scoring Logic ... (positive and negative signals with points)                         |
| Alert System                  | Sticky Note          | Describes Slack alerting system           | -                          | -                                          | ## 🔔 Slack Alerts ... (alert types and threading)                                         |
| Setup Instructions            | Sticky Note          | Configuration steps for Slack and webhook| -                          | -                                          | ## 🔧 Configuration Steps ... (Slack OAuth2, channels, competitors, test webhook)           |
| BeyondPresence Real-Time Webhook | Webhook             | Receives real-time call events            | -                          | Acknowledge Receipt                         |                                                                                             |
| Acknowledge Receipt           | Respond to Webhook    | Sends HTTP 200 acknowledgment             | BeyondPresence Real-Time Webhook | Filter User Messages                      |                                                                                             |
| Filter User Messages          | Switch               | Filters user messages and call end events | Acknowledge Receipt          | Score Sales Opportunity (user_message), Process Call Summary (call_ended) |                                                                                             |
| Score Sales Opportunity       | Code                 | Scores individual user messages           | Filter User Messages (user_message) | Route Alerts                            |                                                                                             |
| Route Alerts                 | Switch               | Routes alerts based on urgency and type   | Score Sales Opportunity      | Slack: Hot Lead Alert, Slack: Competitor Alert, Slack: Qualified Lead |                                                                                             |
| Slack: Hot Lead Alert         | Slack                 | Sends hot lead notifications to Slack     | Route Alerts (hot_lead)      | -                                          |                                                                                             |
| Slack: Competitor Alert       | Slack                 | Sends competitor intelligence alerts      | Route Alerts (competitor_alert) | -                                          |                                                                                             |
| Slack: Qualified Lead         | Slack                 | Sends qualified lead updates               | Route Alerts (qualified_lead) | -                                          |                                                                                             |
| Process Call Summary          | Code                  | Aggregates and scores entire call         | Filter User Messages (call_ended) | Slack: Call Summary                      |                                                                                             |
| Slack: Call Summary          | Slack                 | Posts call summary report to Slack         | Process Call Summary         | -                                          |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `BeyondPresence Real-Time Webhook`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `beyondpresence-sales-intelligence`  
   - Response Mode: Response Node  
   - Options: rawBody disabled (to parse JSON)  

2. **Create Respond to Webhook Node**  
   - Name: `Acknowledge Receipt`  
   - Connect input from webhook node  
   - Response Code: 200  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "status": "success",
       "message": "Sales intelligence processing",
       "timestamp": "{{ $now.toISO() }}"
     }
     ```  

3. **Create Switch Node for Filtering Events**  
   - Name: `Filter User Messages`  
   - Connect input from `Acknowledge Receipt`  
   - Rules:  
     - Output `user_message`: event_type equals `"message"` AND message.sender equals `"user"`  
     - Output `call_ended`: event_type equals `"call_ended"`  
     - Fallback output: none  

4. **Create Code Node for Scoring Sales Opportunity**  
   - Name: `Score Sales Opportunity`  
   - Connect input from `Filter User Messages` output `user_message`  
   - Paste JavaScript scoring logic that:  
     - Extracts message text and metadata  
     - Detects buying/negative signals, competitor mentions  
     - Calculates opportunity score, lead temperature, alerts  
     - Estimates deal size and recommended actions  
   - Ensure error handling returns informative JSON on failure  

5. **Create Switch Node to Route Alerts**  
   - Name: `Route Alerts`  
   - Connect input from `Score Sales Opportunity`  
   - Rules:  
     - `hot_lead`: alertUrgency equals `"high"`  
     - `competitor_alert`: hasCompetitorMention is true  
     - `qualified_lead`: opportunityScore >= 40  
   - Fallback output: none  

6. **Create Slack Node for Hot Lead Alert**  
   - Name: `Slack: Hot Lead Alert`  
   - Connect input from `Route Alerts` output `hot_lead`  
   - Authentication: Slack OAuth2 credentials  
   - Channel: `#sales-hot-leads`  
   - Message: Rich markdown including lead info, score, detected signals, recommendations, deal value  

7. **Create Slack Node for Competitor Alert**  
   - Name: `Slack: Competitor Alert`  
   - Connect input from `Route Alerts` output `competitor_alert`  
   - Authentication: Slack OAuth2  
   - Channel: `#sales-competitors`  
   - Message: Includes competitor mentions, lead info, and response recommendations  

8. **Create Slack Node for Qualified Lead**  
   - Name: `Slack: Qualified Lead`  
   - Connect input from `Route Alerts` output `qualified_lead`  
   - Authentication: Slack OAuth2  
   - Channel: `#sales-qualified`  
   - Message: Lead info, positive buying signals, next steps  

9. **Create Code Node for Processing Call Summary**  
   - Name: `Process Call Summary`  
   - Connect input from `Filter User Messages` output `call_ended`  
   - Script to aggregate all user messages:  
     - Sum scores for buying and negative keywords  
     - Detect competitor mentions  
     - Calculate average score and qualification level  
     - Generate recommendations and next best actions  

10. **Create Slack Node for Call Summary Report**  
    - Name: `Slack: Call Summary`  
    - Connect input from `Process Call Summary`  
    - Authentication: Slack OAuth2  
    - Channel: `#sales-summaries`  
    - Message: Summary report with call duration, scores, signals, competitors, next steps  

11. **Create Sticky Notes (Optional but Recommended)**  
    - Workflow Overview: Describe workflow purpose and setup requirements  
    - Scoring Rules: Document buying and negative signals with points  
    - Alert System: Explain Slack alert logic and threading  
    - Setup Instructions: Provide OAuth2 setup, channel updates, competitor customization, webhook test tips  

12. **Credential Setup**  
    - Create Slack OAuth2 credential in n8n  
    - Ensure permissions include posting to channels: `#sales-hot-leads`, `#sales-competitors`, `#sales-qualified`, `#sales-summaries`  
    - Test Slack connection before enabling workflow  

13. **Testing**  
    - Use webhook test URL to send sample BeyondPresence event JSON with user message  
    - Verify that message is acknowledged, scored, and routed correctly  
    - Confirm Slack alerts appear in correct channels  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                         |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Workflow employs detailed scoring and signal detection logic implemented in JavaScript code nodes.            | Core intellectual property; requires maintenance with evolving sales signals |
| Slack OAuth2 credentials must be granted appropriate channel write permissions to avoid authentication errors | n8n Credential Management, Slack API documentation                      |
| Competitor keywords list is customizable but requires manual updates to remain relevant                       | Customize inside "Score Sales Opportunity" code node                   |
| Regex for deal value extraction supports $K and $M notation, but unusual formats may cause parsing errors     | May require refinement for international currencies                    |
| Slack messages use Markdown formatting with dynamic template expressions for rich notifications               | Slack API message formatting guide                                     |
| Workflow designed for real-time streaming but can be extended for batch or offline processing                 | Consider performance and rate limits when scaling                      |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected data. All processed data is legal and publicly accessible.

---