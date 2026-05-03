Handle Failed Payment Renewals with AI Analysis, Jira Tickets and Slack Alerts

https://n8nworkflows.xyz/workflows/handle-failed-payment-renewals-with-ai-analysis--jira-tickets-and-slack-alerts-11601


# Handle Failed Payment Renewals with AI Analysis, Jira Tickets and Slack Alerts

### 1. Workflow Overview

This workflow automates the handling of failed payment renewal notifications by integrating AI-driven risk analysis, issue tracking, and team notifications. Its primary goal is to triage failed payments, assess churn risk and urgency using OpenAI, create prioritized Jira tickets for follow-up, and alert the team via Slack.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception & Validation**: Receives webhook POST requests containing failed payment data and validates essential fields.
- **1.2 AI Processing & Decision Logic**: Uses OpenAI to analyze the failure event, determine churn risk, urgency, draft recovery emails, and routes high-value failures for higher priority.
- **1.3 Success Path**: Creates Jira tickets based on AI output and configured priority and sends notifications to Slack.
- **1.4 Error Path**: Handles invalid payloads by sending Slack alerts about missing data fields.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
This block receives incoming failed payment notifications via a webhook, then validates the payload to ensure all required information is present before further processing.

**Nodes Involved:**  
- Webhook - Payment Failed  
- Validate Payload  
- Payload Valid? (If node)  
- Slack Error Alert (error notification)

**Node Details:**

- **Webhook - Payment Failed**  
  - *Type:* Webhook  
  - *Role:* Entry point accepting POST requests at `/payment-failed-renewal`.  
  - *Configuration:* HTTP POST, no additional options set.  
  - *Connections:* Output to Validate Payload node.  
  - *Edge Cases:*  
    - Missing or malformed payloads will be caught downstream.  
    - Webhook authentication or security is not shown; may be an integration consideration.

- **Validate Payload**  
  - *Type:* Function  
  - *Role:* Checks that required fields `customerId`, `customerEmail`, `subscriptionId`, and `amount` are present and non-empty.  
  - *Configuration:* Custom JavaScript function iterates over required fields, collects missing ones, and adds validation flags (`validationStatus`, `missingFields`, `hasError`) to output JSON.  
  - *Key Variables:* `$json` input, outputs augmented JSON with validation info.  
  - *Connections:* Output to Payload Valid? node.  
  - *Edge Cases:*  
    - Partial data may cause failures downstream if not caught here.  
    - Function errors could occur if input JSON structure differs.

- **Payload Valid?**  
  - *Type:* If node  
  - *Role:* Branches workflow based on presence of errors detected by Validate Payload.  
  - *Configuration:* Checks boolean condition on `$json.hasError`.  
  - *Connections:*  
    - True branch (error) to Slack Error Alert.  
    - False branch (valid) to AI Analysis.  
  - *Edge Cases:* Expression evaluation failure if `hasError` is missing.

- **Slack Error Alert**  
  - *Type:* Slack  
  - *Role:* Sends a Slack message to notify team about invalid webhook payloads.  
  - *Configuration:* Message text dynamically lists missing fields with markdown formatting for emphasis.  
  - *Connections:* No outputs; terminal error notification.  
  - *Edge Cases:* Slack API failures, authentication errors.

---

#### 2.2 AI Processing & Decision Logic

**Overview:**  
This block leverages OpenAI to analyze failed payment data, assess churn risk, urgency, and draft a recovery email. It then routes the task based on payment amount to determine ticket priority.

**Nodes Involved:**  
- AI Analysis (OpenAI node)  
- Priority Switch (Switch node)  
- Set High Priority (Set node)  
- Set Standard Priority (Set node)

**Node Details:**

- **AI Analysis**  
  - *Type:* OpenAI (GPT-3.5-turbo)  
  - *Role:* Processes input data to:  
    1. Determine churn risk level (Low, Medium, High).  
    2. Determine urgency (Standard, High).  
    3. Generate a polite recovery email draft.  
  - *Configuration:*  
    - System prompt sets role as financial recovery assistant.  
    - User prompt dynamically injects customer email, amount, and failure reason from JSON.  
    - Expects JSON response with fields: `churnRisk`, `urgency`, `emailDraft`.  
  - *Connections:* Output to Priority Switch.  
  - *Edge Cases:*  
    - API rate limits, auth errors.  
    - Unexpected or malformed AI response.  
    - Timeout or network failures.  
    - Missing or invalid input fields may cause poor AI output.

- **Priority Switch**  
  - *Type:* Switch  
  - *Role:* Routes flow based on whether `amount` is greater than $500.  
  - *Configuration:* Compares `$json.amount` > 500.  
  - *Connections:*  
    - True branch to Set High Priority.  
    - False branch to Set Standard Priority.  
  - *Edge Cases:*  
    - Non-numeric or missing `amount` may cause routing errors.  
    - Expression evaluation errors.

- **Set High Priority**  
  - *Type:* Set  
  - *Role:* Adds `ticketPriority` with value "High" to JSON.  
  - *Connections:* Output to Create Jira Finance Ticket.  
  - *Edge Cases:* None significant.

- **Set Standard Priority**  
  - *Type:* Set  
  - *Role:* Adds `ticketPriority` with value "Medium" to JSON.  
  - *Connections:* Output to Create Jira Finance Ticket.  
  - *Edge Cases:* None significant.

---

#### 2.3 Success Path (Jira Ticket Creation and Slack Notification)

**Overview:**  
Creates a Jira ticket with AI-assessed data and priority labels, then sends a Slack notification containing ticket details and customer info.

**Nodes Involved:**  
- Create Jira Finance Ticket (Jira node)  
- Slack Finance Notification (Slack node)

**Node Details:**

- **Create Jira Finance Ticket**  
  - *Type:* Jira  
  - *Role:* Creates a task in the "FIN" Jira project with AI-enhanced data.  
  - *Configuration:*  
    - Issue Type: Task  
    - Summary: Includes customer email and churn risk.  
    - Labels: Always includes "billing-recovery" plus "urgent" if priority is High, else "routine".  
    - Description: Combines urgency, churn risk, AI-generated email draft, and payment failure details.  
  - *Connections:* Output to Slack Finance Notification.  
  - *Edge Cases:*  
    - Jira API auth or permission errors.  
    - Network issues.  
    - Malformed fields causing Jira rejection.

- **Slack Finance Notification**  
  - *Type:* Slack  
  - *Role:* Notifies team about the failed renewal ticket with urgency, risk, customer, amount, and link to Jira issue.  
  - *Configuration:*  
    - Message uses markdown formatting with dynamic data insertion.  
    - Includes hyperlink to Jira ticket URL from previous node output.  
  - *Connections:* Terminal node.  
  - *Edge Cases:* Slack API errors, message formatting issues.

---

#### 2.4 Error Path

**Overview:**  
Handles cases where the input payload is invalid by sending a Slack alert indicating which required fields are missing.

**Nodes Involved:**  
- Slack Error Alert (already covered above)

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                           | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                          |
|---------------------------|--------------------|-----------------------------------------|-----------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook - Payment Failed  | Webhook            | Entry point for failed payment payload  | (none)                      | Validate Payload               |                                                                                                    |
| Validate Payload          | Function           | Validates required fields in payload    | Webhook - Payment Failed    | Payload Valid?                 | ## Validation & Routing Validates payload. If valid, passes to AI for analysis.                    |
| Payload Valid?            | If                 | Branches on validation status            | Validate Payload            | AI Analysis, Slack Error Alert | ## Validation & Routing Validates payload. If valid, passes to AI for analysis.                    |
| Slack Error Alert         | Slack              | Sends alert on invalid payload           | Payload Valid?              | (none)                        | ## Error Path                                                                                       |
| AI Analysis              | OpenAI             | Analyzes failure, churn risk, drafts email | Payload Valid?              | Priority Switch               | ## AI & Logic 1. AI Analysis: Determines Churn Risk & drafts email. 2. Switch: Routes High Value (>$500) to High Priority. |
| Priority Switch          | Switch             | Routes based on amount threshold         | AI Analysis                 | Set High Priority, Set Standard Priority | ## AI & Logic 1. AI Analysis: Determines Churn Risk & drafts email. 2. Switch: Routes High Value (>$500) to High Priority. |
| Set High Priority        | Set                | Marks ticket priority as High             | Priority Switch             | Create Jira Finance Ticket    | ## Success Path Creates Jira ticket with AI-enhanced data and notifies Slack.                      |
| Set Standard Priority    | Set                | Marks ticket priority as Medium           | Priority Switch             | Create Jira Finance Ticket    | ## Success Path Creates Jira ticket with AI-enhanced data and notifies Slack.                      |
| Create Jira Finance Ticket| Jira               | Creates Jira task with AI data            | Set High Priority, Set Standard Priority | Slack Finance Notification   | ## Success Path Creates Jira ticket with AI-enhanced data and notifies Slack.                      |
| Slack Finance Notification| Slack              | Sends notification with Jira ticket info | Create Jira Finance Ticket  | (none)                        | ## Success Path Creates Jira ticket with AI-enhanced data and notifies Slack.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook - Payment Failed`  
   - HTTP Method: POST  
   - Path: `/payment-failed-renewal`  
   - No additional authentication set (consider adding if needed).  

2. **Create Function Node for Validation**  
   - Name: `Validate Payload`  
   - Paste this JavaScript code:  
   ```javascript
   const requiredFields = ['customerId', 'customerEmail', 'subscriptionId', 'amount'];
   const missingFields = [];
   const payload = $input.item.json;

   for (const field of requiredFields) {
     if (!payload[field] || payload[field] === '') {
       missingFields.push(field);
     }
   }

   const isValid = missingFields.length === 0;

   return {
     json: {
       ...payload,
       validationStatus: isValid ? 'valid' : 'invalid',
       missingFields: missingFields,
       hasError: !isValid
     }
   };
   ```
   - Connect output of webhook to this node.

3. **Create If Node for Validation Check**  
   - Name: `Payload Valid?`  
   - Condition: Boolean → `{{$json.hasError}}` equals `true`  
   - Connect output of `Validate Payload` to this node.

4. **Create Slack Node for Error Alert**  
   - Name: `Slack Error Alert`  
   - Text:  
   ```
   ⚠️ *Payment Webhook Error*
   Missing fields: {{$json.missingFields.join(', ')}}
   ```  
   - Enable markdown formatting.  
   - Connect "true" (error) output of `Payload Valid?` to this Slack node.

5. **Create OpenAI Node for AI Analysis**  
   - Name: `AI Analysis`  
   - Model: GPT-3.5-turbo  
   - Prompt (messages):  
     - System: "You are a financial recovery assistant."  
     - User:  
     ```
     Analyze this payment failure:
     Customer: {{$json.customerEmail}}
     Amount: ${{$json.amount}}
     Reason: {{$json.failureReason}}

     1. Determine Churn Risk (Low, Medium, High).
     2. Determine Urgency (Standard, High).
     3. Draft a polite recovery email to the customer.

     Return JSON: { "churnRisk": "...", "urgency": "...", "emailDraft": "..." }
     ```  
   - Connect "false" (valid) output of `Payload Valid?` to this node.

6. **Create Switch Node for Priority Routing**  
   - Name: `Priority Switch`  
   - Add rule: Check if `amount` > 500  
   - Expression for value1: `{{$json.amount}}`  
   - Connect output of `AI Analysis` to this node.

7. **Create Set Node for High Priority**  
   - Name: `Set High Priority`  
   - Set field `ticketPriority` = `"High"`  
   - Connect "true" output of `Priority Switch` to this node.

8. **Create Set Node for Standard Priority**  
   - Name: `Set Standard Priority`  
   - Set field `ticketPriority` = `"Medium"`  
   - Connect "false" output of `Priority Switch` to this node.

9. **Create Jira Node to Create Ticket**  
   - Name: `Create Jira Finance Ticket`  
   - Project Key: `FIN`  
   - Issue Type: `Task`  
   - Summary:  
     ```
     Failed Renewal - {{$json.customerEmail}} (Risk: {{$json.churnRisk}})
     ```  
   - Labels:  
     ```
     billing-recovery
     {{$json.ticketPriority == 'High' ? 'urgent' : 'routine'}}
     ```  
   - Description:  
     ```
     **Analysis**
     - Urgency: {{$json.ticketPriority}}
     - Churn Risk: {{$json.churnRisk}}

     **Draft Email:**
     {{$json.emailDraft}}

     **Details**
     - Customer: {{$json.customerEmail}}
     - Amount: ${{$json.amount}}
     - Reason: {{$json.failureReason}}
     ```  
   - Connect outputs of both `Set High Priority` and `Set Standard Priority` nodes into this node.

10. **Create Slack Node for Success Notification**  
    - Name: `Slack Finance Notification`  
    - Text:  
    ```
    🚨 *Failed Subscription Renewal*

    *Urgency:* {{$json.ticketPriority}}
    *Risk:* {{$json.churnRisk}}

    *Customer:* {{$json.customerEmail}}
    *Amount:* ${{$json.amount}}

    <{{$node['Create Jira Finance Ticket'].json.self}}|View Jira Ticket>
    ```  
    - Enable markdown formatting.  
    - Connect output of `Create Jira Finance Ticket` to this node.

11. **Credential Setup:**  
    - Configure OpenAI credentials with API key for `AI Analysis` node.  
    - Configure Jira credentials with appropriate permissions for `Create Jira Finance Ticket`.  
    - Configure Slack credentials for Slack nodes.

12. **Activate Webhook and Test:**  
    - Deploy workflow.  
    - Test by sending POST requests to `/payment-failed-renewal` with JSON including required fields.  
    - Verify behavior on valid and invalid payloads.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The workflow includes AI integration with OpenAI GPT-3.5-turbo for advanced financial recovery assistance and communication drafting. | AI & Logic Section.                                                                                               |
| Jira tickets are labeled for billing recovery and prioritized based on payment amount exceeding $500 threshold.                      | Jira Ticket Creation.                                                                                             |
| Slack messages use markdown formatting to enhance readability and include dynamic links to Jira tickets for quick access.           | Slack Notification nodes.                                                                                         |
| Setup steps include configuring webhook path, Jira, Slack, OpenAI credentials, and testing with POST requests.                      | Main Overview sticky note content.                                                                                |
| Consider adding security measures to webhook (authentication, IP whitelist) for production environments.                            | Security best practices (not explicitly implemented in this workflow).                                           |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.