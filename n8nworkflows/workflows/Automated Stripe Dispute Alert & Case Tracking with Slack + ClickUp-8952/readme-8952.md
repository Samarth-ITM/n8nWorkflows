Automated Stripe Dispute Alert & Case Tracking with Slack + ClickUp

https://n8nworkflows.xyz/workflows/automated-stripe-dispute-alert---case-tracking-with-slack---clickup-8952


# Automated Stripe Dispute Alert & Case Tracking with Slack + ClickUp

---

### 1. Workflow Overview

This workflow automates the monitoring and management of payment disputes from Stripe. It fetches current disputes, evaluates their priority based on status and deadlines, and alerts the team via Slack while creating corresponding tasks in ClickUp for tracking and resolution.

**Target Use Cases:**  
- Businesses using Stripe for payments who want automated dispute handling  
- Teams needing structured notifications and task creation for dispute management  
- Reducing manual monitoring and speeding up dispute response actions  

**Logical Blocks:**  
- **1.1 Trigger & Fetch:** Manual trigger initiates the workflow and fetches disputes from Stripe API.  
- **1.2 Data Validation:** Checks if disputes exist in the fetched data to prevent errors and unnecessary processing.  
- **1.3 Data Formatting:** Processes raw dispute data into human-readable formats, calculates priority, deadlines, and prepares enriched data.  
- **1.4 Priority Determination:** Classifies disputes into high or standard priority based on status and urgency.  
- **1.5 Notification & Task Creation:** Sends Slack alerts and creates ClickUp tasks aligned with the priority level.  
- **1.6 No Disputes Handling:** If no disputes exist, sends a status summary notification confirming monitoring operation.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Fetch

- **Overview:** Starts the workflow manually and retrieves all current payment disputes from Stripe.  
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - 1️⃣ Fetch Stripe Disputes (HTTP Request)  
  - 📋 Workflow Overview (Sticky Note)  

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually.  
    - Config: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connects to "1️⃣ Fetch Stripe Disputes"  
    - Edge Cases: Manual execution may be missed if no schedule is set.  

  - **1️⃣ Fetch Stripe Disputes**  
    - Type: HTTP Request  
    - Role: Fetches disputes from Stripe’s API endpoint (`/v1/disputes`).  
    - Config: Uses Stripe API credentials preconfigured in n8n; no additional query parameters.  
    - Inputs: Trigger from manual node  
    - Outputs: JSON data of disputes to validation node  
    - Edge Cases: API authentication errors, network timeouts, empty or malformed responses.  
    - Version: HTTP Request node v4.2  
    - Credentials: Stripe API OAuth or API key (configured under Stripe account).  

  - **📋 Workflow Overview**  
    - Type: Sticky Note  
    - Role: Documentation block describing workflow purpose and trigger.  
    - No inputs or outputs.

---

#### 1.2 Data Validation

- **Overview:** Checks if the Stripe API returned any disputes and controls workflow branching accordingly.  
- **Nodes Involved:**  
  - 2️⃣ Validate Disputes Data (If)  
  - 📊 Data Check Info (Sticky Note)  
  - 6️⃣ Send Status Summary (Slack)  
  - ✅ No Action Needed (Sticky Note)  

- **Node Details:**  

  - **2️⃣ Validate Disputes Data**  
    - Type: If node  
    - Role: Checks if the fetched disputes array exists and contains at least one dispute.  
    - Config: Condition evaluates `{{$json.data && $json.data.length > 0}}` equals true.  
    - Inputs: Output from Fetch Stripe Disputes  
    - Outputs:  
      - True branch → Format Dispute Data node  
      - False branch → Send Status Summary node  
    - Edge Cases: Empty array, missing `data` key, or API changes affecting response structure.  

  - **6️⃣ Send Status Summary**  
    - Type: Slack node  
    - Role: Sends informational message when no disputes are found to confirm monitoring status.  
    - Config: Sends text with total disputes found (0), timestamp, and positive status note.  
    - Inputs: False output of validation node  
    - Outputs: None (end of false branch)  
    - Edge Cases: Slack authentication failure, message formatting errors.  
    - Credentials: Slack API OAuth token configured.  

  - **📊 Data Check Info** and **✅ No Action Needed**  
    - Type: Sticky Notes  
    - Role: Document the logic and purpose of checking for dispute data existence and no dispute scenarios.

---

#### 1.3 Data Formatting

- **Overview:** Converts raw Stripe dispute data into enriched, human-readable format including amount formatting, priority calculation, deadline parsing, and customer info extraction.  
- **Nodes Involved:**  
  - 🔧 Format Dispute Data (Code)  

- **Node Details:**  

  - **🔧 Format Dispute Data**  
    - Type: Code (JavaScript) node  
    - Role: Parses the Stripe dispute data from the input, formats amount/currency, calculates days until evidence deadline, assigns priority level (High, Medium, Low), and extracts identifiers and customer info.  
    - Key expressions/logic:  
      - Converts amount from cents to formatted string with currency symbol.  
      - Computes days until dispute evidence deadline compared to current date.  
      - Priority logic: High if deadline ≤3 days or amount ≥ $500; Medium if deadline ≤7 days or amount ≥ $100; else Low.  
      - Extracts email or customer id if available.  
    - Inputs: True branch from Data Validation node (disputes data)  
    - Outputs: Array of enriched dispute JSON objects, each representing one dispute.  
    - Edge Cases: Missing fields in Stripe data, invalid timestamps, zero or negative deadlines.  
    - Version: Code node v2  

---

#### 1.4 Priority Determination

- **Overview:** Analyzes formatted dispute data to branch into high or standard priority workflows based on dispute status.  
- **Nodes Involved:**  
  - 3️⃣ Determine Priority Level (If)  
  - ⚠️ Priority Logic (Sticky Note)  

- **Node Details:**  

  - **3️⃣ Determine Priority Level**  
    - Type: If node  
    - Role: Checks if the dispute status equals "needs_response" to identify high priority cases.  
    - Config: Condition checks `{{$json.data[0].status}} === 'needs_response'`  
    - Inputs: Output from Format Dispute Data node  
    - Outputs:  
      - True → Urgent Slack Alert branch  
      - False → Standard Slack Alert branch  
    - Edge Cases: Status field missing or unknown status values.  

  - **⚠️ Priority Logic**  
    - Type: Sticky Note  
    - Describes criteria for high priority disputes and resulting workflow consequences.

---

#### 1.5 Notification & Task Creation

- **Overview:** Sends Slack notifications and creates ClickUp tasks based on dispute priority level, ensuring team awareness and actionable tracking.  
- **Nodes Involved:**  
  - High Priority Path:  
    - 4a️⃣ Send Urgent Slack Alert (Slack)  
    - 5a️⃣ Create Urgent ClickUp Task (ClickUp)  
    - 🚨 Urgent Actions (Sticky Note)  
  - Standard Priority Path:  
    - 4b️⃣ Send Standard Slack Alert (Slack)  
    - 5b️⃣ Create Standard ClickUp Task (ClickUp)  
    - 📊 Standard Process (Sticky Note)  
  - ✅ Workflow Complete (Sticky Note)  

- **Node Details:**  

  - **4a️⃣ Send Urgent Slack Alert**  
    - Type: Slack node  
    - Role: Sends immediate, prominently formatted Slack alert with full dispute details for urgent response.  
    - Config: Message includes amount, reason, status, priority, customer info, timeline, and IDs.  
    - Inputs: True output from Priority Level node  
    - Outputs: Connects to "5a️⃣ Create Urgent ClickUp Task"  
    - Credentials: Slack API with bot permissions  
    - Edge Cases: Slack API rate limits, message formatting issues.  

  - **5a️⃣ Create Urgent ClickUp Task**  
    - Type: ClickUp node  
    - Role: Creates high-priority task in ClickUp with dispute info, due date set to evidence deadline, and status Open.  
    - Config: Task name with dispute ID and amount, priority set to 1 (highest), due date from dispute data.  
    - Inputs: Output from urgent Slack alert node  
    - Credentials: ClickUp API token configured  
    - Edge Cases: API auth errors, invalid due date formats.  

  - **4b️⃣ Send Standard Slack Alert**  
    - Type: Slack node  
    - Role: Sends standard dispute notification for non-urgent cases, including essential dispute details for tracking.  
    - Config: Message includes amount, reason, priority, evidence due date, customer info, and dispute ID.  
    - Inputs: False output from Priority Level node  
    - Outputs: Connects to "5b️⃣ Create Standard ClickUp Task"  
    - Credentials: Slack API token  
    - Edge Cases: Same as urgent Slack node.  

  - **5b️⃣ Create Standard ClickUp Task**  
    - Type: ClickUp node  
    - Role: Creates task in ClickUp with priority based on calculated priority level (High=1, Medium=2, Low=3), status To Do, and due date.  
    - Config: Task name includes dispute ID and amount, priority dynamically assigned, due date set.  
    - Inputs: Output from standard Slack alert  
    - Credentials: ClickUp API token  
    - Edge Cases: API errors, incorrect priority mapping.  

  - **🚨 Urgent Actions**, **📊 Standard Process**, **✅ Workflow Complete**  
    - Sticky Notes documenting the respective workflow paths and next steps.

---

#### 1.6 No Disputes Handling

- Covered in Block 1.2 Data Validation with Slack summary and sticky notes.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                                      | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                                                          |
|-------------------------------|--------------------|-----------------------------------------------------|-------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Initiates the workflow manually                      | None                          | 1️⃣ Fetch Stripe Disputes         |                                                                                                                                     |
| 📋 Workflow Overview           | Sticky Note        | Describes workflow purpose and start trigger        | None                          | None                             | ## 🔄 WORKFLOW START<br>**Purpose:** This workflow automatically monitors Stripe for payment disputes and creates alerts + tasks.  |
| 1️⃣ Fetch Stripe Disputes       | HTTP Request       | Retrieves all current payment disputes from Stripe  | When clicking ‘Execute workflow’ | 2️⃣ Validate Disputes Data        | Retrieves all current payment disputes from your Stripe account. This includes disputes that customers have filed against payments. |
| 📊 Data Check Info              | Sticky Note        | Explains data existence validation                   | None                          | None                             | ## 🔍 DATA VALIDATION<br>Checks if any disputes were found in the API response to decide further processing.                         |
| 2️⃣ Validate Disputes Data      | If                 | Checks if disputes exist in fetched data             | 1️⃣ Fetch Stripe Disputes       | 🔧 Format Dispute Data, 6️⃣ Send Status Summary | Checks if the Stripe API returned any disputes. If none, sends summary notification and ends.                                       |
| 6️⃣ Send Status Summary         | Slack              | Sends confirmation message if no disputes found     | 2️⃣ Validate Disputes Data (false) | None                             | Sends confirmation message when no disputes need processing. Provides audit trail of workflow execution.                             |
| ✅ No Action Needed             | Sticky Note        | Documents no dispute scenario handling               | None                          | None                             | ## ℹ️ NO DISPUTES FOUND<br>Confirms monitoring is working and no disputes returned.                                                  |
| 🔧 Format Dispute Data          | Code               | Formats raw dispute data and calculates priority     | 2️⃣ Validate Disputes Data (true) | 3️⃣ Determine Priority Level     | Transforms raw Stripe data into user-friendly format, calculates priority, formats amounts, and prepares data for notifications.    |
| 3️⃣ Determine Priority Level    | If                 | Branches workflow based on dispute urgency           | 🔧 Format Dispute Data          | 4a️⃣ Send Urgent Slack Alert, 4b️⃣ Send Standard Slack Alert | Analyzes dispute urgency; high priority disputes get immediate alerts, others get standard notifications.                            |
| ⚠️ Priority Logic              | Sticky Note        | Details criteria for priority assessment              | None                          | None                             | ## ⚡ PRIORITY ASSESSMENT<br>Determines if disputes need immediate attention based on status and deadlines.                          |
| 4a️⃣ Send Urgent Slack Alert    | Slack              | Sends high priority dispute alert to Slack           | 3️⃣ Determine Priority Level (true) | 5a️⃣ Create Urgent ClickUp Task | Sends immediate alert to Slack with comprehensive dispute details, formatted for quick action.                                       |
| 5a️⃣ Create Urgent ClickUp Task | ClickUp            | Creates urgent priority task in ClickUp              | 4a️⃣ Send Urgent Slack Alert   | None                             | Creates high-priority task in ClickUp with detailed action plan and evidence deadline as due date.                                   |
| 4b️⃣ Send Standard Slack Alert  | Slack              | Sends standard dispute notification to Slack        | 3️⃣ Determine Priority Level (false) | 5b️⃣ Create Standard ClickUp Task | Sends standard dispute notification to Slack for non-urgent cases, contains essential information.                                  |
| 5b️⃣ Create Standard ClickUp Task | ClickUp            | Creates standard priority task in ClickUp            | 4b️⃣ Send Standard Slack Alert | None                             | Creates standard priority task with appropriate priority level and comprehensive details.                                            |
| 🚨 Urgent Actions              | Sticky Note        | Describes urgent action workflow path                 | None                          | None                             | ## 🚨 HIGH PRIORITY PATH<br>Trigger and actions for urgent disputes requiring immediate attention.                                    |
| 📊 Standard Process           | Sticky Note        | Describes standard priority workflow                  | None                          | None                             | ## 📋 REGULAR PRIORITY PATH<br>Handles non-urgent disputes with notifications and follow-up tasks.                                  |
| ✅ Workflow Complete           | Sticky Note        | Summarizes workflow end state and next steps          | None                          | None                             | ## 🔄 WORKFLOW COMPLETION<br>Summary of actions taken and recommended next steps for dispute management.                             |
| ⚙️ Setup Instructions          | Sticky Note        | Provides configuration and usage instructions         | None                          | None                             | ## 🚀 AUTOMATION SETUP<br>Instructions for setting up API credentials, Slack, ClickUp, and scheduling.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position at start. No parameters needed.  

2. **Add HTTP Request Node to Fetch Stripe Disputes**  
   - Type: HTTP Request  
   - URL: `https://api.stripe.com/v1/disputes`  
   - Authentication: Predefined credentials using Stripe API OAuth or API key  
   - Connect Manual Trigger → HTTP Request  

3. **Add If Node to Validate Disputes Data**  
   - Type: If  
   - Condition: `{{$json.data && $json.data.length > 0}}` equals `true`  
   - Connect HTTP Request → If Node  

4. **Add Slack Node for Status Summary (False Branch)**  
   - Type: Slack  
   - Text:  
     ```
     ℹ️ **Dispute Monitoring Summary**
     • **Total disputes found:** {{ $json.data.length }}
     • **Time:** {{ new Date().toLocaleString() }}
     • **Status:** No new disputes to process

     All current disputes have been previously handled.

     ✅ Monitoring system is working correctly.
     ```  
   - Credentials: Slack API OAuth token with bot permissions  
   - Connect If Node (False) → Slack Status Summary  

5. **Add Code Node to Format Dispute Data (True Branch)**  
   - Type: Code (JavaScript)  
   - Paste the transformation script:  
     - Extract first dispute from array  
     - Format amount and currency  
     - Calculate days until evidence deadline  
     - Assign priority: High (≤3 days or ≥500 USD), Medium (≤7 days or ≥100 USD), else Low  
     - Extract customer email or ID  
     - Format created and evidence deadline dates  
     - Output enriched JSON object for each dispute  
   - Connect If Node (True) → Code Node  

6. **Add If Node to Determine Priority Level**  
   - Type: If  
   - Condition: `{{$json.status}} === 'needs_response'`  
   - Connect Code Node → Priority If Node  

7. **Add Slack Node for Urgent Alert (True Branch)**  
   - Type: Slack  
   - Message:  
     ```
     🚨 **HIGH PRIORITY DISPUTE ALERT** 🚨

     *Dispute Details:*
     • **Amount:** {{ $json.formatted_amount }}
     • **Reason:** {{ $json.reason.charAt(0).toUpperCase() + $json.reason.slice(1) }}
     • **Status:** {{ $json.status.charAt(0).toUpperCase() + $json.status.slice(1) }}
     • **Priority:** {{ $json.priority }}
     • **Customer:** {{ $json.customer_info }}

     *Timeline:*
     • **Created:** {{ $json.created_date }}
     • **Evidence Due:** {{ $json.evidence_deadline }}
     • **Days Remaining:** {{ $json.days_until_deadline }} days

     *IDs for Reference:*
     • **Dispute ID:** `{{ $json.dispute_id }}`
     • **Charge ID:** `{{ $json.charge_id }}`
     • **Payment Intent:** `{{ $json.payment_intent_id }}`

     ⚠️ Immediate action required for High Priority disputes!
     ```  
   - Credentials: Slack API token  
   - Connect Priority If Node (True) → Urgent Slack Alert  

8. **Add ClickUp Node to Create Urgent Task**  
   - Type: ClickUp  
   - Team, Space, List IDs: set according to your ClickUp workspace  
   - Task Name: `🚨 URGENT: Dispute {{ $json.dispute_id }} - {{ $json.formatted_amount }}`  
   - Status: Open  
   - Due Date: `{{ $json.evidence_deadline }}`  
   - Priority: 1 (highest)  
   - Connect Urgent Slack Alert → ClickUp Urgent Task  

9. **Add Slack Node for Standard Alert (False Branch)**  
   - Type: Slack  
   - Message:  
     ```
     📋 **New Dispute Notification**

     *Details:*
     • **Amount:** {{ $json.formatted_amount }}
     • **Reason:** {{ $json.reason.charAt(0).toUpperCase() + $json.reason.slice(1) }}
     • **Priority:** {{ $json.priority }}
     • **Evidence Due:** {{ $json.evidence_deadline }} ({{ $json.days_until_deadline }} days)
     • **Customer:** {{ $json.customer_info }}

     *Reference:* `{{ $json.dispute_id }}`
     ```  
   - Credentials: Slack API token  
   - Connect Priority If Node (False) → Standard Slack Alert  

10. **Add ClickUp Node to Create Standard Task**  
    - Type: ClickUp  
    - Team, Space, List IDs: same as urgent task but can be different if desired  
    - Task Name: `Dispute: {{ $json.dispute_id }} - {{ $json.formatted_amount }}`  
    - Status: To Do  
    - Due Date: `{{ $json.evidence_deadline }}`  
    - Priority: Use expression:  
      ```
      {{$json.priority === 'High' ? '1' : ($json.priority === 'Medium' ? '2' : '3')}}
      ```  
    - Connect Standard Slack Alert → ClickUp Standard Task  

11. **Add Sticky Notes as Documentation**  
    - Add sticky notes describing workflow overview, data validation, priority logic, urgent actions, standard process, no disputes found, workflow completion, and setup instructions at appropriate places for clarity.

12. **Credentials Setup**  
    - Stripe: Add API key or OAuth credentials with permission to read disputes.  
    - Slack: Add bot OAuth token with permission to post messages in desired channels.  
    - ClickUp: Add API token with permission to create tasks in specified workspace and lists.

13. **Testing & Scheduling**  
    - Test workflow manually.  
    - Set up a schedule trigger (e.g., every 4 hours) if automation without manual trigger is desired.  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                        |
|---------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| **Setup Instructions:** Requires Stripe API access, Slack bot permissions, and ClickUp API token.       | Included in ⚙️ Setup Instructions sticky note.                                                                         |
| Scheduling the workflow every 4 hours is recommended for timely dispute monitoring.                      | Suggested in ⚙️ Setup Instructions sticky note.                                                                         |
| Slack messages use user mentions by ID for visibility; ensure the user ID is valid in your workspace.    | Note in Slack nodes configuration.                                                                                     |
| Priority logic is customizable in the code node to adjust thresholds for dispute urgency.               | See 🔧 Format Dispute Data node code comments.                                                                          |
| ClickUp tasks use specific list/team/space IDs; these must be replaced with your workspace identifiers. | ClickUp API documentation: https://clickup.com/api                                                                      |
| Stripe dispute API reference: https://stripe.com/docs/api/disputes/list                                   |                                                                                                                       |
| Slack API message formatting guide: https://api.slack.com/messaging/composing                            |                                                                                                                       |

---

This comprehensive reference document covers the full structure, logic, and configuration details needed to understand, reproduce, and maintain the "Automated Stripe Dispute Alert & Case Tracking with Slack + ClickUp" workflow. It anticipates error cases such as missing disputes, API failures, and authentication issues, providing a robust integration for dispute management automation.