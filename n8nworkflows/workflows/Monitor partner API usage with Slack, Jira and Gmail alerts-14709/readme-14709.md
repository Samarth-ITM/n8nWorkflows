Monitor partner API usage with Slack, Jira and Gmail alerts

https://n8nworkflows.xyz/workflows/monitor-partner-api-usage-with-slack--jira-and-gmail-alerts-14709


# Monitor partner API usage with Slack, Jira and Gmail alerts

# 1. Workflow Overview

This workflow monitors partner API consumption received through an HTTP webhook and triggers internal alerts when usage reaches important thresholds. It validates incoming payloads, calculates quota utilization, and then branches into different escalation paths using Slack, Jira, and Gmail.

Its main use case is partner quota monitoring for internal operations teams. Depending on the usage percentage, the workflow can send an early Slack warning, create a Jira task for review, or trigger a full critical escalation with email, Jira, and Slack.

## 1.1 Input Reception and Payload Validation
The workflow starts with a webhook that receives partner usage data. A Code node validates the required fields and checks that numeric values are usable before allowing the payload to continue.

## 1.2 Usage Calculation and Threshold Routing
Once validated, another Code node computes the usage percentage from `consumed / quota`. A Switch node then routes the item into one of several threshold-based branches.

## 1.3 90% Review Escalation Branch
If usage is at or above 90%, the workflow creates a Jira review ticket and then posts a Slack message referencing that ticket.

## 1.4 80% Early Warning Branch
If usage is at or above 80%, the workflow sends a Slack early warning to internal teams.

## 1.5 100% Critical Escalation Branch
If usage is at or above 100%, the workflow sends an email, creates a Jira ticket, and posts a Slack escalation.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Payload Validation

**Overview:**  
This block accepts incoming POST requests and validates whether the payload contains the required partner usage fields. Only items marked as valid continue into usage analysis.

**Nodes Involved:**  
- Incoming Partner Usage Data
- Validate Partner Usage Payload
- Is Usage Payload Valid?

### Node Details

#### Incoming Partner Usage Data
- **Type and technical role:** `n8n-nodes-base.webhook`; entry-point trigger node receiving external HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `a25d3181-6c3f-4014-bf85-ff07294f6ccc`
  - No special response handling options configured
- **Key expressions or variables used:** None in the node itself.
- **Input and output connections:**
  - Input: none
  - Output: `Validate Partner Usage Payload`
- **Version-specific requirements:** Uses webhook node version `2.1`.
- **Edge cases or potential failure types:**
  - Incorrect HTTP method
  - Requests sent to the wrong path
  - Missing `body` structure expected by the next Code node
- **Sub-workflow reference:** None.

#### Validate Partner Usage Payload
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript validation layer.
- **Configuration choices:**
  - Reads input from `items[0].json.body`
  - Checks for:
    - `partner_id`
    - `partner_name`
    - `quota`
    - `consumed`
  - Validates `quota` and `consumed` as numeric
  - Returns one item with:
    - `status: "invalid"` and an `errors` array if validation fails
    - `status: "valid"` if validation succeeds
  - Spreads original payload fields into output
- **Key expressions or variables used:**
  - `const data = items[0].json.body;`
  - `errors.push(...)`
  - `status`
- **Input and output connections:**
  - Input: `Incoming Partner Usage Data`
  - Output: `Is Usage Payload Valid?`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - If `items[0].json.body` is undefined, the script may fail before validation completes
  - The node only validates presence and numeric convertibility, not semantic correctness such as negative values
  - `plan` and `timestamp` are used later but are not validated here
- **Sub-workflow reference:** None.

#### Is Usage Payload Valid?
- **Type and technical role:** `n8n-nodes-base.if`; routing gate based on validation result.
- **Configuration choices:**
  - Strict comparison of `{{$json.status}}` to `valid`
  - Only the true output is connected
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: `Validate Partner Usage Payload`
  - True output: `Calculate API Usage Percentage`
  - False output: unconnected
- **Version-specific requirements:** IF node version `2.2` with conditions version `2`.
- **Edge cases or potential failure types:**
  - Invalid payloads are silently dropped because the false branch is not connected
  - If `status` is missing, the item will also be dropped
- **Sub-workflow reference:** None.

---

## 2.2 Usage Calculation and Threshold Routing

**Overview:**  
This block computes the usage percentage and decides whether alerting should happen. It then routes high-usage items into threshold-specific branches.

**Nodes Involved:**  
- Calculate API Usage Percentage
- Switch

### Node Details

#### Calculate API Usage Percentage
- **Type and technical role:** `n8n-nodes-base.code`; business logic node for utilization computation.
- **Configuration choices:**
  - Converts `quota` and `consumed` to numbers
  - Returns `action: "invalid_usage"` if values are not usable
  - Computes `usage_pct` rounded to 2 decimals
  - Returns `action: "no_alert"` if usage is below 80%
  - Returns `action: "proceed"` if usage is 80% or higher
- **Key expressions or variables used:**
  - `const quota = Number(item.quota);`
  - `const consumed = Number(item.consumed);`
  - `const usage_pct = Math.round((consumed / quota) * 100 * 100) / 100;`
- **Input and output connections:**
  - Input: `Is Usage Payload Valid?`
  - Output: `Switch`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - `if (!quota || isNaN(quota) || isNaN(consumed))` treats `quota = 0` as invalid, which is probably intentional but should be noted
  - Negative values are not blocked
  - `action` is set but never used by downstream routing
  - Items below 80% still go to the Switch node
- **Sub-workflow reference:** None.

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`; threshold router.
- **Configuration choices:**
  - Output 0: `usage_pct >= 90`
  - Output 1: `usage_pct >= 80`
  - Output 2: `usage_pct >= 100`
- **Key expressions or variables used:**
  - `={{ $json.usage_pct }}`
- **Input and output connections:**
  - Input: `Calculate API Usage Percentage`
  - Output 0: `Create Jira Partner Review Ticket`
  - Output 1: `Send Slack Partner Usage Alert1`
  - Output 2: `Send a message`
- **Version-specific requirements:** Switch version `3.3`.
- **Edge cases or potential failure types:**
  - The branch ordering creates overlapping threshold behavior concerns
  - Since `>= 80`, `>= 90`, and `>= 100` overlap, actual execution depends on n8n Switch behavior and output matching mode
  - The workflow comments describe 80%, 90%, and 100% paths, but this configuration may not behave exactly as intended if only the first match is used
  - Items below 80% are effectively dropped because there is no fallback/default branch connected
- **Sub-workflow reference:** None.

**Important logic note:**  
The sticky notes describe distinct threshold handling:
- 80% = Slack
- 90% = Jira + Slack
- 100% = Email + Jira + Slack

However, because the rules are configured in the order `>= 90`, `>= 80`, `>= 100`, the 100% rule may never be reached in a first-match routing setup. This is a design inconsistency and should be corrected during reproduction.

---

## 2.3 90% Review Escalation Branch

**Overview:**  
This branch handles high usage requiring internal follow-up. It first creates a Jira task and then publishes a Slack alert containing the issue key.

**Nodes Involved:**  
- Create Jira Partner Review Ticket
- Send Slack Partner Usage Alert

### Node Details

#### Create Jira Partner Review Ticket
- **Type and technical role:** `n8n-nodes-base.jira`; creates a Jira issue for internal review.
- **Configuration choices:**
  - Project: `10000` (`n8n sample project`)
  - Issue type: `10003` (`Task`)
  - Summary includes partner name and rounded usage percentage
  - Description includes:
    - partner name
    - partner ID
    - plan
    - quota
    - consumed
    - usage percentage
    - timestamp
    - recommended review action
- **Key expressions or variables used:**
  - `{{$json["partner_name"]}}`
  - `{{Math.round($json["usage_pct"])}}`
  - `{{$json["plan"]}}`
  - `{{$json["timestamp"]}}`
- **Input and output connections:**
  - Input: `Switch` output 0
  - Output: `Send Slack Partner Usage Alert`
- **Version-specific requirements:** Jira node version `1`.
- **Edge cases or potential failure types:**
  - Missing Jira credentials
  - Invalid project or issue type IDs
  - Jira field validation failures
  - `plan` and `timestamp` may be empty because they are not enforced upstream
- **Sub-workflow reference:** None.

#### Send Slack Partner Usage Alert
- **Type and technical role:** `n8n-nodes-base.slack`; sends a formatted Slack channel message.
- **Configuration choices:**
  - Posts to channel ID `C09S57E2JQ2` (`n8n`)
  - Message includes:
    - partner name from `Validate Partner Usage Payload`
    - rounded usage percentage from current item
    - plan and timestamp from `Validate Partner Usage Payload`
    - Jira issue key from current Jira node output
- **Key expressions or variables used:**
  - `$('Validate Partner Usage Payload').item.json.partner_name`
  - `Math.round($json["usage_pct"])`
  - `$json.key`
- **Input and output connections:**
  - Input: `Create Jira Partner Review Ticket`
  - Output: none
- **Version-specific requirements:** Slack node version `2.3`.
- **Edge cases or potential failure types:**
  - Missing Slack credentials
  - Invalid channel selection or bot permissions
  - Cross-node expression references can fail if execution context changes unexpectedly
  - If Jira creation fails, this node never runs
- **Sub-workflow reference:** None.

---

## 2.4 80% Early Warning Branch

**Overview:**  
This branch provides a non-ticketing early warning when usage crosses the lower threshold. It sends a Slack alert to encourage proactive monitoring.

**Nodes Involved:**  
- Send Slack Partner Usage Alert1

### Node Details

#### Send Slack Partner Usage Alert1
- **Type and technical role:** `n8n-nodes-base.slack`; sends an early warning Slack message.
- **Configuration choices:**
  - Posts to channel ID `C09S57E2JQ2`
  - Message includes partner name, rounded usage percentage, plan, timestamp, and recommended action
  - No Jira reference is included
- **Key expressions or variables used:**
  - `$('Validate Partner Usage Payload').item.json.partner_name`
  - `Math.round($json["usage_pct"])`
  - `$('Validate Partner Usage Payload').item.json.plan`
  - `$('Validate Partner Usage Payload').item.json.timestamp`
- **Input and output connections:**
  - Input: `Switch` output 1
  - Output: none
- **Version-specific requirements:** Slack node version `2.3`.
- **Edge cases or potential failure types:**
  - Missing Slack credentials
  - Invalid channel permissions
  - If the Switch sends 90% or 100% traffic here due to rule overlap, this branch may fire more often than intended
- **Sub-workflow reference:** None.

---

## 2.5 100% Critical Escalation Branch

**Overview:**  
This branch handles the most severe case, where partner usage reaches or exceeds full quota. It sends an email first, then creates a Jira issue, then posts a Slack escalation.

**Nodes Involved:**  
- Send a message
- Create Jira Partner Review Ticket1
- Send Slack Partner Usage Alert2

### Node Details

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`; sends a critical HTML email.
- **Configuration choices:**
  - Subject references partner name and 100% usage limit
  - HTML body includes partner details, quota metrics, timestamp, and urgent recommended actions
  - `sendTo` is currently empty
- **Key expressions or variables used:**
  - `{{$json["partner_name"]}}`
  - `{{$json["partner_id"]}}`
  - `{{$json["plan"]}}`
  - `{{$json["quota"]}}`
  - `{{$json["consumed"]}}`
  - `{{Math.round($json["usage_pct"])}}`
  - `{{$json["timestamp"]}}`
- **Input and output connections:**
  - Input: `Switch` output 2
  - Output: `Create Jira Partner Review Ticket1`
- **Version-specific requirements:** Gmail node version `2.1`.
- **Edge cases or potential failure types:**
  - Missing Gmail OAuth2 credentials
  - Empty `sendTo` means the node is incomplete and will fail until configured
  - HTML content uses expression syntax extensively; malformed values may affect rendering
- **Sub-workflow reference:** None.

#### Create Jira Partner Review Ticket1
- **Type and technical role:** `n8n-nodes-base.jira`; creates a Jira ticket after critical email dispatch.
- **Configuration choices:**
  - Same project and issue type as the 90% branch
  - Summary and description draw values from the `Switch` node using explicit node references
- **Key expressions or variables used:**
  - `$('Switch').item.json.partner_name`
  - `$('Switch').item.json.usage_pct`
  - `$('Switch').item.json.partner_id`
  - `$('Switch').item.json.plan`
  - `$('Switch').item.json.quota`
  - `$('Switch').item.json.consumed`
  - `$('Switch').item.json.timestamp`
- **Input and output connections:**
  - Input: `Send a message`
  - Output: `Send Slack Partner Usage Alert2`
- **Version-specific requirements:** Jira node version `1`.
- **Edge cases or potential failure types:**
  - Missing Jira credentials
  - Field validation errors in Jira
  - Use of `$('Switch')` may be fragile if multiple items or branch ambiguity arise
- **Sub-workflow reference:** None.

#### Send Slack Partner Usage Alert2
- **Type and technical role:** `n8n-nodes-base.slack`; sends final critical Slack notification after Jira creation.
- **Configuration choices:**
  - Posts to channel `C09S57E2JQ2`
  - Includes partner name, usage percentage, plan, timestamp, and Jira issue key
- **Key expressions or variables used:**
  - `$('Validate Partner Usage Payload').item.json.partner_name`
  - `Math.round($json["usage_pct"])`
  - `$('Validate Partner Usage Payload').item.json.plan`
  - `$('Validate Partner Usage Payload').item.json.timestamp`
  - `$json.key`
- **Input and output connections:**
  - Input: `Create Jira Partner Review Ticket1`
  - Output: none
- **Version-specific requirements:** Slack node version `2.3`.
- **Edge cases or potential failure types:**
  - Missing Slack credentials
  - Invalid channel permissions
  - Since current item is Jira output, `usage_pct` may not exist unless Jira preserves input fields in output shape as expected
  - Safer expressions would reference the earlier source item explicitly
- **Sub-workflow reference:** None.

---

## 2.6 Documentation / Annotation Nodes

**Overview:**  
These sticky notes document the intended logic and setup guidance directly on the canvas. They are not executable but are important for understanding the visual design intent.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for the webhook and validation area.
- **Configuration choices:** Describes incoming partner usage data and payload validation purpose.
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** Sticky note for usage analysis block.
- **Configuration choices:** Describes percentage calculation and threshold checking.
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** Sticky note for 90% escalation area.
- **Configuration choices:** Describes Jira + Slack internal review trigger once usage crosses 90%.
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** Sticky note for 100% escalation area.
- **Configuration choices:** Describes email + Jira + Slack critical handling.
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** General overview note.
- **Configuration choices:** Describes intended flow and setup steps, including a Switch node for thresholds.
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** Sticky note for 80% warning branch.
- **Configuration choices:** Describes Slack-only early warning behavior.
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Incoming Partner Usage Data | Webhook | Receives external partner usage payload via POST |  | Validate Partner Usage Payload | ## Incoming Partner Usage Data<br>This section receives incoming partner API usage data through a webhook and validates the payload structure and values. It ensures only correctly formatted and meaningful usage data is allowed to move forward in the workflow. |
| Validate Partner Usage Payload | Code | Validates required fields and numeric values | Incoming Partner Usage Data | Is Usage Payload Valid? | ## Incoming Partner Usage Data<br>This section receives incoming partner API usage data through a webhook and validates the payload structure and values. It ensures only correctly formatted and meaningful usage data is allowed to move forward in the workflow. |
| Is Usage Payload Valid? | If | Filters valid vs invalid payloads | Validate Partner Usage Payload | Calculate API Usage Percentage | ## Incoming Partner Usage Data<br>This section receives incoming partner API usage data through a webhook and validates the payload structure and values. It ensures only correctly formatted and meaningful usage data is allowed to move forward in the workflow. |
| Calculate API Usage Percentage | Code | Computes usage percentage and alert intent | Is Usage Payload Valid? | Switch | ## Analyze API Usage Levels<br>This section calculates the partner’s API usage percentage based on allowed limits and checks whether the usage has crossed the defined alert threshold. It decides whether the situation requires internal review or no action. |
| Switch | Switch | Routes items by usage threshold | Calculate API Usage Percentage | Create Jira Partner Review Ticket; Send Slack Partner Usage Alert1; Send a message | ## Analyze API Usage Levels<br>This section calculates the partner’s API usage percentage based on allowed limits and checks whether the usage has crossed the defined alert threshold. It decides whether the situation requires internal review or no action. |
| Create Jira Partner Review Ticket | Jira | Creates review ticket for high usage | Switch | Send Slack Partner Usage Alert | ## 90% Usage Alert – Internal Review Trigger<br>When API usage crosses 90% of the allocated quota, this section escalates the alert by creating a Jira ticket along with a Slack notification. It ensures internal teams initiate a formal review process and prepare for potential partner communication or plan adjustments. |
| Send Slack Partner Usage Alert | Slack | Posts Slack alert after Jira ticket creation | Create Jira Partner Review Ticket |  | ## 90% Usage Alert – Internal Review Trigger<br>When API usage crosses 90% of the allocated quota, this section escalates the alert by creating a Jira ticket along with a Slack notification. It ensures internal teams initiate a formal review process and prepare for potential partner communication or plan adjustments. |
| Send Slack Partner Usage Alert1 | Slack | Sends early warning Slack message | Switch |  | ## 80% Usage Alert – Early Warning Notification<br>This section monitors when a partner reaches 80% of their API usage quota. It acts as an early warning system by sending a Slack notification to internal teams, enabling proactive monitoring and giving sufficient time to assess usage trends before critical limits are reached. |
| Send a message | Gmail | Sends critical escalation email | Switch | Create Jira Partner Review Ticket1 | ## 100% Usage Alert – Critical Escalation & Action Required<br>This section handles critical scenarios where a partner fully exhausts their API quota. It triggers immediate escalation by sending an email, creating a Jira ticket and notifying via Slack, ensuring rapid response to prevent service disruption and initiate urgent upgrade or mitigation actions. |
| Create Jira Partner Review Ticket1 | Jira | Creates Jira ticket for critical usage | Send a message | Send Slack Partner Usage Alert2 | ## 100% Usage Alert – Critical Escalation & Action Required<br>This section handles critical scenarios where a partner fully exhausts their API quota. It triggers immediate escalation by sending an email, creating a Jira ticket and notifying via Slack, ensuring rapid response to prevent service disruption and initiate urgent upgrade or mitigation actions. |
| Send Slack Partner Usage Alert2 | Slack | Posts final Slack alert for critical usage | Create Jira Partner Review Ticket1 |  | ## 100% Usage Alert – Critical Escalation & Action Required<br>This section handles critical scenarios where a partner fully exhausts their API quota. It triggers immediate escalation by sending an email, creating a Jira ticket and notifying via Slack, ensuring rapid response to prevent service disruption and initiate urgent upgrade or mitigation actions. |
| Sticky Note | Sticky Note | Visual documentation for input/validation block |  |  |  |
| Sticky Note1 | Sticky Note | Visual documentation for analysis block |  |  |  |
| Sticky Note2 | Sticky Note | Visual documentation for 90% branch |  |  |  |
| Sticky Note3 | Sticky Note | Visual documentation for 100% branch |  |  |  |
| Sticky Note4 | Sticky Note | General workflow explanation and setup notes |  |  |  |
| Sticky Note5 | Sticky Note | Visual documentation for 80% branch |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**
   - Name it **Incoming Partner Usage Data**
   - Set method to **POST**
   - Set path to `a25d3181-6c3f-4014-bf85-ff07294f6ccc`
   - Keep response options default unless you want custom webhook responses

2. **Create a Code node for payload validation**
   - Name it **Validate Partner Usage Payload**
   - Connect it after the webhook
   - Paste JavaScript equivalent to:
     - Read `items[0].json.body`
     - Verify `partner_id`, `partner_name`, `quota`, and `consumed`
     - Check that `quota` and `consumed` are numeric
     - Return `status: "valid"` or `status: "invalid"` plus `errors`
   - Recommended improvement: also validate `plan` and `timestamp` if they are operationally required later

3. **Create an IF node**
   - Name it **Is Usage Payload Valid?**
   - Connect it after the validation Code node
   - Add one condition:
     - Left value: `={{ $json.status }}`
     - Operator: equals
     - Right value: `valid`
   - Connect only the **true** output forward
   - Optionally add a false branch to log or respond to invalid submissions

4. **Create a Code node for usage calculation**
   - Name it **Calculate API Usage Percentage**
   - Connect it from the true output of the IF node
   - Add JavaScript logic to:
     - Convert `quota` and `consumed` to numbers
     - Return invalid marker if numbers are not usable
     - Compute `usage_pct = consumed / quota * 100`
     - Round to 2 decimals
     - Mark items below 80 as `no_alert`
     - Mark items at or above 80 as `proceed`
   - Recommended improvement: stop execution for `< 80` instead of letting those items reach the Switch

5. **Create a Switch node**
   - Name it **Switch**
   - Connect it after the calculation node
   - Configure usage-based rules
   - To match the intended business logic reliably, configure in this order:
     1. `usage_pct >= 100`
     2. `usage_pct >= 90`
     3. `usage_pct >= 80`
   - This order is recommended because the provided workflow places `>= 100` after `>= 90` and `>= 80`, which can make the 100% branch unreachable depending on matching behavior
   - Connect:
     - 100% output to Gmail branch
     - 90% output to Jira branch
     - 80% output to Slack-only branch

6. **Create the 80% Slack alert node**
   - Add a Slack node named **Send Slack Partner Usage Alert1**
   - Connect it from the 80% output of the Switch
   - Configure:
     - Resource/action for sending a channel message
     - Channel: `C09S57E2JQ2` or your target channel
     - Message text including:
       - partner name
       - rounded usage percentage
       - plan
       - timestamp
       - recommended action
   - Add Slack API credentials

7. **Create the 90% Jira node**
   - Add a Jira node named **Create Jira Partner Review Ticket**
   - Connect it from the 90% output of the Switch
   - Configure:
     - Project: `10000` or your own Jira project
     - Issue type: `Task` / ID `10003`
     - Summary: partner name + usage %
     - Description: partner details, usage, timestamp, recommended action
   - Add Jira Software Cloud credentials

8. **Create the 90% Slack notification node**
   - Add a Slack node named **Send Slack Partner Usage Alert**
   - Connect it after the 90% Jira node
   - Configure a message that includes:
     - partner name
     - usage %
     - plan
     - timestamp
     - Jira issue key from the Jira node output
   - Add Slack credentials if not already set

9. **Create the 100% Gmail node**
   - Add a Gmail node named **Send a message**
   - Connect it from the 100% output of the Switch
   - Configure:
     - Recipient in **Send To** field
     - Subject indicating urgent 100% quota exhaustion
     - HTML email body containing partner information, usage stats, timestamp, and recommended urgent actions
   - Add Gmail OAuth2 credentials
   - Important: in the provided workflow, `sendTo` is blank; you must fill this for the node to work

10. **Create the 100% Jira node**
    - Add a Jira node named **Create Jira Partner Review Ticket1**
    - Connect it after the Gmail node
    - Configure the same Jira project and issue type as the 90% branch
    - Use summary/description expressions referencing the branch input data
    - Prefer current-item expressions over cross-node references where possible for robustness

11. **Create the 100% Slack node**
    - Add a Slack node named **Send Slack Partner Usage Alert2**
    - Connect it after the critical Jira node
    - Configure a message including:
      - partner name
      - usage %
      - plan
      - timestamp
      - Jira key
    - Recommended improvement: reference original branch data explicitly instead of assuming Jira output still contains `usage_pct`

12. **Add optional sticky notes**
    - Create sticky notes to describe:
      - input and validation
      - usage analysis
      - 80% warning
      - 90% escalation
      - 100% critical escalation
      - overall workflow behavior and setup

13. **Configure credentials**
    - **Slack API**
      - Authorize workspace
      - Ensure the app/bot can post to the target channel
    - **Jira Software Cloud**
      - Configure site, email/API token or OAuth depending on your setup
      - Verify project access and issue creation permission
    - **Gmail OAuth2**
      - Connect the mailbox used for critical alerts
      - Ensure sending permissions are valid

14. **Test with sample payloads**
   - Example scenarios to test:
     - Invalid payload with missing `partner_id`
     - Valid payload with usage below 80%
     - Usage at 80%
     - Usage at 90%
     - Usage at 100% or above
   - Example data shape expected by the workflow body:
     - `partner_id`
     - `partner_name`
     - `plan`
     - `quota`
     - `consumed`
     - `timestamp`

15. **Recommended corrections before production**
   - Add explicit handling for invalid payloads
   - Add a no-op or log path for `< 80%`
   - Reorder the Switch rules to make `>=100` evaluate before `>=90` and `>=80`
   - Validate `plan` and `timestamp`
   - Fill in Gmail recipient
   - Normalize expressions to avoid fragile cross-node references

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How workflow works: Webhook receives partner API usage data (quota, consumed, partner details); Payload is validated to ensure required fields are correct; Usage percentage is calculated from quota vs consumed; Switch node routes flow based on thresholds (80%, 90%, 100%); 80% sends Slack alert for early warning; 90% sends Slack alert + creates Jira ticket for review; 100% sends Slack + Jira + Email for critical escalation; Ensures timely alerts, tracking and action to prevent service issues | General workflow note |
| Setup and Configuration Steps: Create webhook node to accept usage data; Add Code node to validate payload fields; Add Code node to calculate usage percentage; Configure Switch node for 80%, 90%, 100% conditions; Set up Slack node for notifications; Set up Jira node for ticket creation; Set up Gmail node for critical alerts; Add credentials for all integrations (Slack, Jira, Gmail); Test with sample data to verify all scenarios; Deploy workflow after successful testing | General workflow note |

## Additional implementation note
There is a mismatch between the documented behavior and the actual Switch configuration. If reproduced exactly as JSON, the threshold routing may not behave as intended for 100% events. For reliable production use, the threshold logic should be made mutually exclusive or ordered from highest to lowest severity.