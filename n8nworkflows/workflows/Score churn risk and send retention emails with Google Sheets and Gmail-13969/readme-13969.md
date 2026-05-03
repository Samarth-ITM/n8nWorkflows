Score churn risk and send retention emails with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/score-churn-risk-and-send-retention-emails-with-google-sheets-and-gmail-13969


# Score churn risk and send retention emails with Google Sheets and Gmail

# 1. Workflow Overview

This workflow identifies gym or membership customers who may be at risk of churn, classifies them by risk level, sends alerts for the most urgent cases, and logs the resulting actions in Google Sheets.

Typical use cases include:
- Daily retention monitoring for gyms and fitness studios
- Proactive intervention before membership cancellation
- Operational reporting for retention managers
- Automated prioritization of outreach based on behavioral and payment signals

The workflow is organized into four main logical blocks:

## 1.1 Member Data Intake
The workflow starts manually, then reads member records from a Google Sheet containing customer and activity data.

## 1.2 Churn Scoring Engine
A Code node computes derived metrics such as days since last visit, days to renewal, visit variation, and churn risk score. It also assigns each member a `risk_level`, `main_trigger`, and `value_at_risk`.

## 1.3 Risk-Based Routing
A Switch node splits records into four branches: `critical`, `high`, `medium`, and `low`.

## 1.4 Intervention and Logging
- Critical-risk members trigger an internal Gmail alert and are logged in a retention sheet.
- High-risk members trigger a winback-style Gmail notification and are logged.
- Medium-risk members are logged for later follow-up.
- Low-risk members are ignored via a No Operation node.

There is only one workflow entry point in the provided JSON:
- **Manual Trigger**: `When clicking ‘Execute workflow’`

There are no sub-workflows or Execute Workflow nodes in this workflow.

---

# 2. Block-by-Block Analysis

## 2.1 Block A — Member Data Intake

### Overview
This block starts the workflow and loads the member dataset from Google Sheets. In the template, execution is manual, but the note indicates this is intended to be replaced by a scheduled trigger in production.

### Nodes Involved
- `When clicking ‘Execute workflow’`
- `Fetch Member Data`

### Node Details

#### 2.1.1 When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point used for testing or on-demand runs.
- **Configuration choices:**  
  No parameters are configured; it simply starts the workflow when manually executed in the editor.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Fetch Member Data`
- **Version-specific requirements:**  
  Type version `1`; standard manual trigger behavior.
- **Edge cases or potential failure types:**  
  - No runtime failure expected
  - Not suitable for unattended production execution
- **Sub-workflow reference:**  
  None.

#### 2.1.2 Fetch Member Data
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads member rows from a Google Sheet.
- **Configuration choices:**  
  - Google Sheets credential: `Google Sheets account CHR`
  - Document: sample spreadsheet `AI Membership Retention & Churn Prevention System`
  - Sheet: `Members_Database` (`gid=0`)
  - Operation is implicitly the standard row-read mode for this node setup
- **Key expressions or variables used:**  
  No custom expressions are defined in the main parameters.
- **Input and output connections:**  
  - Input: `When clicking ‘Execute workflow’`
  - Output: `Calculate Churn Ris`
- **Version-specific requirements:**  
  Type version `4.7`; this matters because Google Sheets node UI and options differ across versions.
- **Edge cases or potential failure types:**  
  - Google OAuth credential missing or expired
  - Spreadsheet access denied
  - Sheet renamed or deleted
  - Unexpected column names causing downstream code issues
  - Empty sheet leading to zero output items
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block B — Churn Scoring Engine

### Overview
This block performs the core churn evaluation. It normalizes incoming member data, parses dates, computes engagement and payment risk, and produces classification fields consumed by later routing and messaging nodes.

### Nodes Involved
- `Calculate Churn Ris`

### Node Details

#### 2.2.1 Calculate Churn Ris
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that scores each incoming member record.
- **Configuration choices:**  
  The node uses custom JavaScript to:
  - Parse date fields from multiple formats
  - Compute elapsed time metrics
  - Compute visit variation between current and previous month
  - Detect cancelled memberships and failed payments
  - Build an engagement score
  - Map numeric score to `risk_level`
  - Calculate `value_at_risk`
- **Key expressions or variables used:**  
  Important input fields expected from Google Sheets:
  - `checkins_30d`
  - `checkins_prev_month`
  - `cancelled_classes_30d`
  - `equivalent_monthly_price`
  - `payment_status`
  - `membership_status`
  - `last_checkin`
  - `renewal_date`
  - plus identity/context fields such as `name`, `member_id`, `main_center`, `plan`

  Derived output fields:
  - `today_iso`
  - `days_since_last_visit`
  - `days_to_renewal`
  - `visit_variation_pct`
  - `churn_risk_score`
  - `risk_level`
  - `main_trigger`
  - `renewal_bucket`
  - `value_at_risk`

  Internal helper functions:
  - `parseDate(s)`
  - `daysBetween(a, b)`
  - `clamp(n, min, max)`
  - `round(n)`
  - `bucketRenewal(daysToRenewal)`
- **Input and output connections:**  
  - Input: `Fetch Member Data`
  - Output: `Route by Risk Level`
- **Version-specific requirements:**  
  Type version `2`; JavaScript Code node behavior depends on n8n version but this script is standard for current Code nodes.
- **Edge cases or potential failure types:**  
  - Missing fields default to `0` or empty strings in some cases, which may hide data quality issues
  - Invalid date strings return `null`
  - Non-numeric text in numeric columns becomes `NaN`; because `Number(...)` is used directly, malformed sheet values may propagate invalid calculations
  - Timezone assumptions are based on midnight normalization and workflow timezone `Europe/Madrid`
  - Cancelled members are forcibly classified as `low` with score `0`
  - Payment failures override all other logic and force `critical` with score `95`
- **Sub-workflow reference:**  
  None.

### Scoring Logic Summary
The scoring logic is:

1. **Cancelled memberships**
   - If `membership_status === 'cancelled'`, assign:
     - `churn_risk_score = 0`
     - `risk_level = 'low'`
     - `main_trigger = 'cancelled'`
     - `value_at_risk = 0`

2. **Payment failure**
   - If `payment_status === 'failed'`, assign:
     - `churn_risk_score = 95`
     - `risk_level = 'critical'`
     - `main_trigger = 'payment_failed'`
     - `value_at_risk = monthly revenue`

3. **Engagement risk**
   - Inactivity:
     - `>= 30 days`: +45
     - `>= 14 days`: +30
     - `>= 7 days`: +15
   - Low current usage:
     - `<= 2` check-ins: +20
     - `<= 4` check-ins: +10
   - Usage drop versus previous month:
     - `<= -50%`: +30
     - `<= -30%`: +20
   - Cancelled classes:
     - `>= 3`: +20
     - `= 2`: +12
     - `= 1`: +6
   - Engagement score capped at `85`

4. **Risk thresholds**
   - `>= 90`: `critical`
   - `>= 70`: `high`
   - `>= 45`: `medium`
   - otherwise: `low`

5. **Value at risk**
   - Only populated for `high` and `critical` records

---

## 2.3 Block C — Risk-Based Routing

### Overview
This block routes each scored member into one of four downstream actions according to `risk_level`. It is the decision gate between scoring and intervention.

### Nodes Involved
- `Route by Risk Level`

### Node Details

#### 2.3.1 Route by Risk Level
- **Type and technical role:** `n8n-nodes-base.switch`  
  Multi-branch conditional router.
- **Configuration choices:**  
  Four named outputs are configured using strict string equality on `{{$json.risk_level}}`:
  - `CRITICAL` when `risk_level = critical`
  - `HIGH` when `risk_level = high`
  - `MEDIUM` when `risk_level = medium`
  - `LOW` when `risk_level = low`
- **Key expressions or variables used:**  
  - `={{$json.risk_level}}`
- **Input and output connections:**  
  - Input: `Calculate Churn Ris`
  - Outputs:
    - `CRITICAL` → `Prepare Critical Alert`
    - `HIGH` → `Prepare High Risk Email`
    - `MEDIUM` → `Format Medium Risk Log`
    - `LOW` → `Ignore Low Risk`
- **Version-specific requirements:**  
  Type version `3.3`; switch rule configuration differs in older versions.
- **Edge cases or potential failure types:**  
  - If `risk_level` is missing or contains unexpected casing/spelling, the item may not match any output
  - Because strict validation is enabled, type mismatches can prevent routing
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block D — Critical Risk Handling

### Overview
This branch formats a critical alert, sends an internal email notification, and logs the action to the retention sheet.

### Nodes Involved
- `Prepare Critical Alert`
- `Send Critical Notification`
- `Log Critical Action`

### Node Details

#### 2.4.1 Prepare Critical Alert
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds a formatted text field for the critical message while preserving all existing fields.
- **Configuration choices:**  
  - Adds `message` as a string field
  - `includeOtherFields = true`
- **Key expressions or variables used:**  
  The message includes:
  - `{{$json.main_trigger}}`
  - `{{$json.days_since_last_visit}}`
  - `{{$json.days_to_renewal}}`
  - `{{$json.value_at_risk}}`
- **Input and output connections:**  
  - Input: `Route by Risk Level` (critical branch)
  - Output: `Send Critical Notification`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - If expected fields are null, the generated message will contain blank or literal null-like values
- **Sub-workflow reference:**  
  None.

#### 2.4.2 Send Critical Notification
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a plain-text Gmail email for urgent intervention.
- **Configuration choices:**  
  - Gmail credential: `Gmail account CHR`
  - Recipient: hardcoded placeholder `your email here`
  - Subject dynamically includes member name and center
  - Message uses both current item data and explicit references to `Route by Risk Level`
  - `appendAttribution = false`
  - `emailType = text`
- **Key expressions or variables used:**  
  Subject:
  - `{{ $('Route by Risk Level').item.json.name }}`
  - `{{ $('Route by Risk Level').item.json.main_center }}`

  Body:
  - `{{ $('Route by Risk Level').item.json.name }}`
  - `{{ $('Route by Risk Level').item.json.main_center }}`
  - `{{ $('Route by Risk Level').item.json.plan }}`
  - `{{ $('Route by Risk Level').item.json.risk_level }}`
  - `{{ $('Route by Risk Level').item.json.payment_status }}`
  - `{{ $('Route by Risk Level').item.json.renewal_date }}`
  - `{{ $('Route by Risk Level').item.json.days_to_renewal }}`
  - `{{ $json.message }}`
- **Input and output connections:**  
  - Input: `Prepare Critical Alert`
  - Output: `Log Critical Action`
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Gmail OAuth issues
  - Invalid or missing recipient address
  - Quota/rate limits
  - Cross-node item reference may behave unexpectedly if item linking breaks in future edits
- **Sub-workflow reference:**  
  None.

#### 2.4.3 Log Critical Action
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a retention log row after a critical notification is sent.
- **Configuration choices:**  
  - Operation: `append`
  - Document: same sample spreadsheet
  - Sheet: `Retention_Log`
  - Uses explicit column mapping
  - `useAppend = true`
- **Key expressions or variables used:**  
  - `date = {{$now.format(('dd/LL/yyyy')) }}`
  - `name = {{ $('Calculate Churn Ris').item.json.name }}`
  - `action = critical_alert_sent`
  - `trigger = {{ $('Calculate Churn Ris').item.json.main_trigger }}`
  - `member_id = {{ $('Calculate Churn Ris').item.json.member_id }}`
  - `risk_level = {{ $('Calculate Churn Ris').item.json.risk_level }}`
  - `risk_value = {{ $('Calculate Churn Ris').item.json.value_at_risk }}`
- **Input and output connections:**  
  - Input: `Send Critical Notification`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - Sheet permissions or missing tab
  - Column schema drift in Google Sheets
  - Date formatting depends on n8n expression support
  - Logging occurs only if email sending succeeded, since it is downstream of Gmail
- **Sub-workflow reference:**  
  None.

---

## 2.5 Block E — High Risk Handling

### Overview
This branch prepares a high-risk retention message, sends an internal or operational winback-style alert, and logs the action.

### Nodes Involved
- `Prepare High Risk Email`
- `Send Winback Email`
- `Log High Risk Action`

### Node Details

#### 2.5.1 Prepare High Risk Email
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds a formatted explanatory message for high-risk members.
- **Configuration choices:**  
  - Adds field `mensaje_winback`
  - Preserves existing fields with `includeOtherFields = true`
- **Key expressions or variables used:**  
  - `{{$json.main_trigger}}`
  - `{{$json.visit_variation_pct}}`
  - `{{$json.days_since_last_visit}}`
  - `{{$json.days_to_renewal}}`
  - `{{$json.value_at_risk}}`
- **Input and output connections:**  
  - Input: `Route by Risk Level` (high branch)
  - Output: `Send Winback Email`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Null fields may produce degraded but still valid message text
  - Field naming inconsistency exists with downstream node
- **Sub-workflow reference:**  
  None.

#### 2.5.2 Send Winback Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a plain-text high-risk member alert.
- **Configuration choices:**  
  - Gmail credential: `Gmail account CHR`
  - Recipient: expression returning placeholder `your email here`
  - Subject includes member name and center
  - Plain-text email
  - `appendAttribution = false`
- **Key expressions or variables used:**  
  Subject:
  - `{{ $json.name }}`
  - `{{ $json.main_center }}`

  Body includes:
  - `{{ $json.name }}`
  - `{{ $json.main_center }}`
  - `{{ $json.plan }}`
  - `{{ $json.risk_level }}`
  - `{{ $json.checkins_30d }}`
  - `{{ $json.visit_variation_pct }}`
  - `{{ $json.days_since_last_visit }}`
  - `{{ $json.winback_message }}`
- **Input and output connections:**  
  - Input: `Prepare High Risk Email`
  - Output: `Log High Risk Action`
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - **Important configuration bug:** upstream node creates `mensaje_winback`, but this Gmail node references `winback_message`
  - As written, the “Detail” section will likely be empty unless the field is renamed or the expression is corrected
  - Gmail auth, quota, or invalid email recipient errors
- **Sub-workflow reference:**  
  None.

#### 2.5.3 Log High Risk Action
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row indicating that a high-risk winback action was sent.
- **Configuration choices:**  
  - Operation: `append`
  - Target sheet: `Retention_Log`
  - Explicit column mapping
- **Key expressions or variables used:**  
  - `date = {{$now.format(('dd/LL/yyyy')) }}`
  - `name = {{ $('Calculate Churn Ris').item.json.name }}`
  - `action = winback_sent`
  - `trigger = {{ $('Calculate Churn Ris').item.json.main_trigger }}`
  - `member_id = {{ $('Calculate Churn Ris').item.json.member_id }}`
  - `risk_level = {{ $('Calculate Churn Ris').item.json.risk_level }}`
  - `risk_value = {{ $('Calculate Churn Ris').item.json.value_at_risk }}`
- **Input and output connections:**  
  - Input: `Send Winback Email`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  Same as other Google Sheets append nodes; additionally, this log only occurs after successful email sending.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Block F — Medium Risk Handling

### Overview
This branch does not send email. It formats an advisory note and writes the member to the retention log for softer follow-up.

### Nodes Involved
- `Format Medium Risk Log`
- `Log Medium Risk Member`

### Node Details

#### 2.6.1 Format Medium Risk Log
- **Type and technical role:** `n8n-nodes-base.set`  
  Prepares a medium-risk note while retaining all original data.
- **Configuration choices:**  
  - Adds `mensaje_winback`
  - `includeOtherFields = true`
- **Key expressions or variables used:**  
  - `{{$json.main_trigger}}`
  - `{{$json.days_since_last_visit}}`
  - `{{$json.visit_variation_pct}}`
- **Input and output connections:**  
  - Input: `Route by Risk Level` (medium branch)
  - Output: `Log Medium Risk Member`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Generated field is not used downstream, so formatting problems have no operational effect in this version
- **Sub-workflow reference:**  
  None.

#### 2.6.2 Log Medium Risk Member
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row for medium-risk follow-up.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet: `Retention_Log`
  - Explicit column mapping
- **Key expressions or variables used:**  
  - `date = {{$now.format(('dd/LL/yyyy')) }}`
  - `name = {{ $('Calculate Churn Ris').item.json.name }}`
  - `action = scheduled_followup`
  - `trigger = {{ $('Calculate Churn Ris').item.json.main_trigger }}`
  - `member_id = {{ $('Calculate Churn Ris').item.json.member_id }}`
  - `risk_level = {{ $('Calculate Churn Ris').item.json.risk_level }}`
  - `risk_value = {{ $('Calculate Churn Ris').item.json.value_at_risk }}`
- **Input and output connections:**  
  - Input: `Format Medium Risk Log`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - Google Sheets auth/access issues
  - Missing logging sheet or modified headers
- **Sub-workflow reference:**  
  None.

---

## 2.7 Block G — Low Risk Handling

### Overview
This branch deliberately does nothing for low-risk members.

### Nodes Involved
- `Ignore Low Risk`

### Node Details

#### 2.7.1 Ignore Low Risk
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Terminal placeholder node used to explicitly mark the low-risk path as intentionally ignored.
- **Configuration choices:**  
  No parameters.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Route by Risk Level` (low branch)
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  No meaningful execution risk.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start |  | Fetch Member Data | ## A. Member data intake\nThis section starts the workflow and loads the member records that will be evaluated.\nIn the demo version, the workflow is triggered manually. In a real production setup, this step is usually replaced with a scheduled trigger that runs automatically every day. |
| Fetch Member Data | Google Sheets | Read member records from Google Sheets | When clicking ‘Execute workflow’ | Calculate Churn Ris | ## A. Member data intake\nThis section starts the workflow and loads the member records that will be evaluated.\nIn the demo version, the workflow is triggered manually. In a real production setup, this step is usually replaced with a scheduled trigger that runs automatically every day.\n## 📊 Sample Data Sheet\nTo use this template, make a copy of the sample Google Sheet below and connect it to the Google Sheets nodes:\n👉 [Click here to access the sample sheet](https://docs.google.com/spreadsheets/d/1HGWMasSZvefI3wt5KgXlvh4l6WssMHHjNtJABZEKbAk/edit?usp=sharing)\n⚙️ Setup steps\n- Go to **File → Make a copy**\n- Connect your Google account in the Google Sheets nodes\n- Point the nodes to your copied sheet |
| Calculate Churn Ris | Code | Compute churn score, risk level, and derived metrics | Fetch Member Data | Route by Risk Level | ## B. Churn scoring engine\nThis is the decision-making core of the workflow.\nThe code node analyzes each member using predefined retention logic and assigns a churn risk score or category. This score estimates how likely the member is to cancel their membership. |
| Route by Risk Level | Switch | Route members by churn risk level | Calculate Churn Ris | Prepare Critical Alert; Prepare High Risk Email; Format Medium Risk Log; Ignore Low Risk | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Prepare Critical Alert | Set | Format critical-risk alert message | Route by Risk Level | Send Critical Notification | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Send Critical Notification | Gmail | Send urgent internal email alert for critical cases | Prepare Critical Alert | Log Critical Action | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Log Critical Action | Google Sheets | Append critical action log row | Send Critical Notification |  | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Prepare High Risk Email | Set | Format high-risk message content | Route by Risk Level | Send Winback Email | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Send Winback Email | Gmail | Send high-risk retention alert email | Prepare High Risk Email | Log High Risk Action | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Log High Risk Action | Google Sheets | Append high-risk action log row | Send Winback Email |  | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Format Medium Risk Log | Set | Prepare medium-risk follow-up note | Route by Risk Level | Log Medium Risk Member | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Log Medium Risk Member | Google Sheets | Append medium-risk follow-up log row | Format Medium Risk Log |  | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Ignore Low Risk | No Operation | Explicitly ignore low-risk members | Route by Risk Level |  | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Sticky Note | Sticky Note | Documentation note |  |  | ## Who this template is for\nThis workflow is designed for gyms, fitness studios, and membership-based businesses that want to proactively reduce customer churn and improve retention.\nIt is ideal for businesses that manage recurring memberships and want to automatically detect members who may be at risk of canceling, then trigger the appropriate follow-up action based on their churn risk level.\nThis template is especially useful for operations teams, retention managers, and gym owners who want to replace reactive retention efforts with a more systematic and automated process.\nWhat this workflow does\nThis workflow automatically identifies members who may be at risk of canceling their membership and routes them through different retention paths based on their churn risk level.\nThe workflow:\n- Reads member data from a data source\n- Calculates a churn risk score for each member\n- Classifies each member into a retention category\n- Triggers different actions depending on the risk level\n- Sends follow-up communication when needed\n- Logs the outcome for tracking and operational visibility\nThis allows the business to prioritize outreach and intervene before a cancellation happens.\nHow it works\nThe workflow starts by loading member data into n8n. It then passes that data into a churn scoring engine, where each member is evaluated based on predefined business logic.\nThe scoring logic assigns a risk level that represents the likelihood of churn. This risk level is then used by a switch node to route each member into the correct path.\nMembers classified as critical or high risk are sent through a stronger intervention flow, which can include personalized email outreach and retention tracking. Members in medium risk can be logged for softer intervention or later monitoring. Members with low risk do not require immediate action and can be ignored or stored for reporting purposes.\nBy separating members into clearly defined churn categories, the workflow helps retention teams act faster and more consistently. |
| Sticky Note1 | Sticky Note | Documentation note for intake block |  |  | ## A. Member data intake\nThis section starts the workflow and loads the member records that will be evaluated.\nIn the demo version, the workflow is triggered manually. In a real production setup, this step is usually replaced with a scheduled trigger that runs automatically every day. |
| Sticky Note2 | Sticky Note | Documentation note for scoring block |  |  | ## B. Churn scoring engine\nThis is the decision-making core of the workflow.\nThe code node analyzes each member using predefined retention logic and assigns a churn risk score or category. This score estimates how likely the member is to cancel their membership. |
| Sticky Note3 | Sticky Note | Documentation note for routing block |  |  | ## C. Risk-based routing\nThis section uses the churn classification to route each member into the appropriate path.\nDepending on the assigned level, such as critical, high, medium, or low, the workflow determines whether the member should receive immediate intervention, be monitored, or require no action. |
| Sticky Note4 | Sticky Note | Documentation note with sample sheet link |  |  | ## 📊 Sample Data Sheet\nTo use this template, make a copy of the sample Google Sheet below and connect it to the Google Sheets nodes:\n👉 [Click here to access the sample sheet](https://docs.google.com/spreadsheets/d/1HGWMasSZvefI3wt5KgXlvh4l6WssMHHjNtJABZEKbAk/edit?usp=sharing)\n⚙️ Setup steps\n- Go to **File → Make a copy**\n- Connect your Google account in the Google Sheets nodes\n- Point the nodes to your copied sheet |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## 4.1 Prepare external systems
1. Create or copy a Google Sheet for the workflow.
2. Add at least two tabs:
   - `Members_Database`
   - `Retention_Log`
3. In `Members_Database`, include columns expected by the Code node, at minimum:
   - `member_id`
   - `name`
   - `main_center`
   - `plan`
   - `checkins_30d`
   - `checkins_prev_month`
   - `cancelled_classes_30d`
   - `equivalent_monthly_price`
   - `payment_status`
   - `membership_status`
   - `last_checkin`
   - `renewal_date`
4. In `Retention_Log`, create columns:
   - `date`
   - `member_id`
   - `name`
   - `risk_level`
   - `trigger`
   - `action`
   - `risk_value`
5. Create or connect:
   - a **Google Sheets OAuth2** credential
   - a **Gmail OAuth2** credential

## 4.2 Create the trigger
6. Add a **Manual Trigger** node.
7. Name it: **When clicking ‘Execute workflow’**.

## 4.3 Add the member data reader
8. Add a **Google Sheets** node.
9. Name it: **Fetch Member Data**.
10. Connect **When clicking ‘Execute workflow’ → Fetch Member Data**.
11. Configure the Google Sheets credential.
12. Select your spreadsheet document.
13. Select the `Members_Database` sheet.
14. Configure it to read rows from the sheet.

## 4.4 Add the churn scoring node
15. Add a **Code** node.
16. Name it: **Calculate Churn Ris**.
17. Connect **Fetch Member Data → Calculate Churn Ris**.
18. Paste the JavaScript logic that:
   - parses `YYYY-MM-DD` and `dd/MM/yyyy`
   - computes `days_since_last_visit`
   - computes `days_to_renewal`
   - computes `visit_variation_pct`
   - marks cancelled memberships as low risk
   - marks failed payments as critical
   - scores engagement using inactivity, usage, usage drop, and cancellations
   - outputs:
     - `today_iso`
     - `days_since_last_visit`
     - `days_to_renewal`
     - `visit_variation_pct`
     - `churn_risk_score`
     - `risk_level`
     - `main_trigger`
     - `renewal_bucket`
     - `value_at_risk`

### Recommended output contract from the Code node
19. Ensure every item keeps original fields and also includes:
   - `risk_level`
   - `main_trigger`
   - `value_at_risk`

## 4.5 Add the routing node
20. Add a **Switch** node.
21. Name it: **Route by Risk Level**.
22. Connect **Calculate Churn Ris → Route by Risk Level**.
23. Add four outputs with renamed output keys:
   - `CRITICAL`
   - `HIGH`
   - `MEDIUM`
   - `LOW`
24. Configure conditions:
   - Output `CRITICAL`: `{{$json.risk_level}}` equals `critical`
   - Output `HIGH`: `{{$json.risk_level}}` equals `high`
   - Output `MEDIUM`: `{{$json.risk_level}}` equals `medium`
   - Output `LOW`: `{{$json.risk_level}}` equals `low`

## 4.6 Build the critical branch
25. Add a **Set** node.
26. Name it: **Prepare Critical Alert**.
27. Connect the `CRITICAL` output of the Switch to it.
28. Enable keeping other input fields.
29. Add a string field named `message`.
30. Set its value to a formatted multiline alert including:
   - `main_trigger`
   - `days_since_last_visit`
   - `days_to_renewal`
   - `value_at_risk`

31. Add a **Gmail** node.
32. Name it: **Send Critical Notification**.
33. Connect **Prepare Critical Alert → Send Critical Notification**.
34. Configure the Gmail credential.
35. Set recipient email to your internal retention or operations email.
36. Set email type to **text**.
37. Disable attribution if desired.
38. Set subject dynamically with member name and center.
39. Set message body with:
   - member name
   - center
   - plan
   - risk level
   - payment status
   - renewal date and days to renewal
   - the `message` field from the Set node
   - recommended action text

40. Add a **Google Sheets** node.
41. Name it: **Log Critical Action**.
42. Connect **Send Critical Notification → Log Critical Action**.
43. Use the same Google Sheets credential.
44. Select the same spreadsheet.
45. Select `Retention_Log`.
46. Set operation to **Append**.
47. Map fields:
   - `date` = current date formatted as `dd/LL/yyyy`
   - `member_id`
   - `name`
   - `risk_level`
   - `trigger`
   - `action` = `critical_alert_sent`
   - `risk_value`

## 4.7 Build the high-risk branch
48. Add a **Set** node.
49. Name it: **Prepare High Risk Email**.
50. Connect the `HIGH` output of the Switch to it.
51. Keep all other fields.
52. Add a string field for the formatted high-risk message.

### Important correction
53. Use a consistent field name. The JSON currently uses `mensaje_winback` in the Set node but `winback_message` in the Gmail node.  
54. To avoid errors, choose one of these two fixes:
   - rename the Set field to `winback_message`, or
   - update the Gmail node to reference `mensaje_winback`

55. Add a **Gmail** node.
56. Name it: **Send Winback Email**.
57. Connect **Prepare High Risk Email → Send Winback Email**.
58. Configure Gmail credential.
59. Set recipient email.
60. Set email type to **text**.
61. Set subject to include member name and center.
62. Set message body to include:
   - member name
   - center
   - plan
   - risk level
   - check-ins in last 30 days
   - usage variation
   - days since last visit
   - the formatted winback message
   - recommended action text

63. Add a **Google Sheets** node.
64. Name it: **Log High Risk Action**.
65. Connect **Send Winback Email → Log High Risk Action**.
66. Set operation to **Append** into `Retention_Log`.
67. Map:
   - `date`
   - `member_id`
   - `name`
   - `risk_level`
   - `trigger`
   - `action` = `winback_sent`
   - `risk_value`

## 4.8 Build the medium-risk branch
68. Add a **Set** node.
69. Name it: **Format Medium Risk Log**.
70. Connect the `MEDIUM` output of the Switch to it.
71. Keep other fields.
72. Add a formatted note field describing:
   - trigger
   - days since last visit
   - visit variation
   - recommendation for soft follow-up

73. Add a **Google Sheets** node.
74. Name it: **Log Medium Risk Member**.
75. Connect **Format Medium Risk Log → Log Medium Risk Member**.
76. Configure append mode to `Retention_Log`.
77. Map:
   - `date`
   - `member_id`
   - `name`
   - `risk_level`
   - `trigger`
   - `action` = `scheduled_followup`
   - `risk_value`

## 4.9 Build the low-risk branch
78. Add a **No Operation** node.
79. Name it: **Ignore Low Risk**.
80. Connect the `LOW` output of the Switch to it.

## 4.10 Optional documentation notes
81. Add sticky notes describing:
   - target business users
   - member data intake
   - churn scoring engine
   - risk-based routing
   - sample sheet setup instructions

## 4.11 Validate credentials and test
82. Test the Google Sheets read node to confirm data is returned.
83. Run the Code node and inspect output fields for several sample members.
84. Verify all four risk paths are reachable using test rows.
85. Send test Gmail messages to a valid mailbox.
86. Confirm `Retention_Log` receives appended rows.
87. Replace placeholder emails such as `your email here` with real recipients.
88. Activate the workflow only after replacing the manual trigger if production automation is required.

### Recommended production improvement
89. Replace the Manual Trigger with a **Schedule Trigger** to run daily.
90. Add validation before the Code node if sheet quality is inconsistent.
91. Consider adding an error branch or Error Trigger workflow for failed Gmail or Sheets operations.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample Google Sheet for this workflow template | [Google Sheet link](https://docs.google.com/spreadsheets/d/1HGWMasSZvefI3wt5KgXlvh4l6WssMHHjNtJABZEKbAk/edit?usp=sharing) |
| Setup instruction: make a copy of the sample sheet before connecting your own Google account | Google Sheets setup |
| Intended audience: gyms, fitness studios, and membership-based businesses focused on churn reduction | Template positioning |
| Suggested production change: replace Manual Trigger with a scheduled trigger that runs every day | Operational deployment note |
| The workflow logs only medium, high, and critical actions; low-risk members are intentionally ignored | Behavior note |
| Configuration issue to fix: `Prepare High Risk Email` creates `mensaje_winback`, while `Send Winback Email` reads `winback_message` | Important implementation note |