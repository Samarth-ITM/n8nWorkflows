Qualify High-Budget Leads: Typeform to HubSpot, Google Sheets & Slack Alerts

https://n8nworkflows.xyz/workflows/qualify-high-budget-leads--typeform-to-hubspot--google-sheets---slack-alerts-8240


# Qualify High-Budget Leads: Typeform to HubSpot, Google Sheets & Slack Alerts

---

### 1. Workflow Overview

This workflow automates the qualification and routing of high-budget leads collected via Typeform, integrating with HubSpot CRM, Google Sheets, and Slack for notifications. It targets businesses that want to prioritize and efficiently manage leads based on budget thresholds and lead sources.

The workflow is logically divided into the following blocks:
- **1.1 Form Intake & Lead Qualification:** Captures new lead submissions from Typeform and evaluates if the lead’s budget exceeds a high-priority threshold ($5,000).
- **1.2 CRM Integration & Priority Handling:** For qualified high-budget leads, creates or updates the contact in HubSpot and adds a priority follow-up task.
- **1.3 Lead Source Routing & Storage:** Determines the lead source (Facebook or SurveyMonkey) and logs lead data into Google Sheets accordingly.
- **1.4 Automated Lead Acknowledgment:** Sends a Slack notification to alert the sales or marketing team about a new high-priority lead.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Intake & Lead Qualification

**Overview:**  
This block receives lead data from a Typeform submission webhook and immediately checks whether the lead’s budget surpasses $5,000 to classify it as high-priority.

**Nodes Involved:**  
- 📋 Typeform Submission Trigger  
- 💰 Check High-Budget Lead

**Node Details:**

- **📋 Typeform Submission Trigger**  
  - *Type:* Typeform Trigger node  
  - *Role:* Listens for new form submissions on a specified Typeform form via webhook.  
  - *Configuration:* Uses a webhook linked to a specified Typeform form ID and authenticated via Typeform API credentials.  
  - *Expressions:* None explicitly; outputs submission JSON.  
  - *Connections:* Outputs to the "💰 Check High-Budget Lead" node.  
  - *Edge Cases:* Webhook misconfiguration, invalid credentials, or Typeform downtime could block submissions.

- **💰 Check High-Budget Lead**  
  - *Type:* If node  
  - *Role:* Evaluates if the “What is your Budget” field from the Typeform JSON exceeds 5000.  
  - *Configuration:* Condition uses a numeric comparison on expression `{{$json['What is your Budget']}} > 5000`.  
  - *Connections:* If true, proceeds to HubSpot contact creation; if false, the lead is ignored (no downstream nodes).  
  - *Edge Cases:* Missing or non-numeric budget values cause condition failure or unexpected behavior.

---

#### 2.2 CRM Integration & Priority Handling

**Overview:**  
Handles qualified leads by creating or updating corresponding contacts in HubSpot with lead details and budget, then adds a high-priority task in HubSpot to prompt sales follow-up.

**Nodes Involved:**  
- 👤 HubSpot — Create/Update Contact  
- 📝 HubSpot — Add Priority Task

**Node Details:**

- **👤 HubSpot — Create/Update Contact**  
  - *Type:* HubSpot node (CRM contact management)  
  - *Role:* Creates or updates a contact in HubSpot based on email, adds first name, phone number, and custom property “Budget”.  
  - *Configuration:*  
    - Email: `{{$json.Email}}`  
    - First Name: `{{$json.Name}}`  
    - Phone Number: `{{$json['Phone Number']}}`  
    - Custom Property “Budget”: `{{$json['What is your Budget']}}`  
  - *Authentication:* HubSpot App Token credentials.  
  - *Connections:* Output connects to “📝 HubSpot — Add Priority Task”.  
  - *Edge Cases:* Missing email or authentication failures can cause node failure.

- **📝 HubSpot — Add Priority Task**  
  - *Type:* HubSpot node (Engagement creation)  
  - *Role:* Adds a task engagement in HubSpot with a body note indicating priority and budget.  
  - *Configuration:*  
    - Engagement Type: task  
    - Body: `"Priority - High (Budget - {{ $json['What is your Budget'] }})"`  
  - *Authentication:* HubSpot App Token credentials.  
  - *Connections:* Proceeds to lead source routing block.  
  - *Edge Cases:* HubSpot API limits or invalid engagement data could cause failures.

---

#### 2.3 Lead Source Routing & Storage

**Overview:**  
Based on the lead source field, routes the lead to Google Sheets logging for Facebook or SurveyMonkey leads. This supports marketing analytics and campaign tracking.

**Nodes Involved:**  
- 📘 Check if Lead is of (Facebook/SurveyMonkey)  
- 📄 Log Lead to Google Sheet

**Node Details:**

- **📘 Check if Lead is of (Facebook/SurveyMonkey)**  
  - *Type:* If node  
  - *Role:* Checks if lead source or lead_source field contains "Facebook" or equals "SurveyMonkey".  
  - *Configuration:* Two string conditions combined with OR:  
    - Contains "Facebook"  
    - Equals "SurveyMonkey"  
  - *Connections:* If true, routes to Google Sheets logging; if false, no further action.  
  - *Edge Cases:* Missing lead_source or source fields cause false negatives; case sensitivity handled by strict mode.

- **📄 Log Lead to Google Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends the lead data as a new row to a specified Google Sheet.  
  - *Configuration:*  
    - Operation: Append row  
    - Spreadsheet ID and Sheet Name are parameterized placeholders.  
    - Columns mapping to lead data not explicitly defined but assumed.  
  - *Authentication:* Google Sheets OAuth2 credentials.  
  - *Connections:* Outputs to Slack notification node.  
  - *Edge Cases:* Incorrect sheet ID or name, permission errors, or API limits may cause failures.

---

#### 2.4 Automated Lead Acknowledgment

**Overview:**  
Sends a Slack message to notify the team about the new high-priority lead, including key details like name, email, phone number, budget, and lead source.

**Nodes Involved:**  
- 😊 Sends Message to Slack

**Node Details:**

- **😊 Sends Message to Slack**  
  - *Type:* Slack node (Webhook or API message send)  
  - *Role:* Posts a message to a specified Slack channel with lead details formatted in text.  
  - *Configuration:*  
    - Text template includes: Name, Email, Number, Budget, Lead source  
    - Channel specified via channel ID and cached channel name.  
  - *Authentication:* Slack API OAuth2 or token credentials.  
  - *Connections:* Terminal node, no outputs.  
  - *Edge Cases:* Invalid channel ID, revoked Slack token, or network issues may cause message send failure.

---

### 3. Summary Table

| Node Name                         | Node Type                   | Functional Role                          | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                   |
|----------------------------------|-----------------------------|----------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| 📋 Typeform Submission Trigger   | Typeform Trigger            | Capture new lead submissions           | -                                | 💰 Check High-Budget Lead              | 📋 Form Intake & Lead Qualification: Captures lead data & evaluates budget > $5,000          |
| 💰 Check High-Budget Lead         | If                         | Check if lead budget is high priority  | 📋 Typeform Submission Trigger   | 👤 HubSpot — Create/Update Contact     | 📋 Form Intake & Lead Qualification                                                         |
| 👤 HubSpot — Create/Update Contact| HubSpot CRM Contact         | Create/update contact with lead info   | 💰 Check High-Budget Lead         | 📝 HubSpot — Add Priority Task         | 👤 CRM Integration & Priority Handling: Add/update contact with budget                       |
| 📝 HubSpot — Add Priority Task    | HubSpot Engagement          | Add priority follow-up task in HubSpot | 👤 HubSpot — Create/Update Contact| 📘 Check if Lead is of (Facebook/SurveyMonkey)| 👤 CRM Integration & Priority Handling                                                    |
| 📘 Check if Lead is of (Facebook/SurveyMonkey) | If              | Route leads based on source             | 📝 HubSpot — Add Priority Task    | 📄 Log Lead to Google Sheet             | 📘/📊 Lead Source Routing & Storage: Routes Facebook & SurveyMonkey leads to Google Sheets  |
| 📄 Log Lead to Google Sheet       | Google Sheets               | Log lead data in Google Sheet           | 📘 Check if Lead is of (Facebook/SurveyMonkey) | 😊 Sends Message to Slack               | 📧 Automated Lead Acknowledgment: Logs lead data to Google Sheets                           |
| 😊 Sends Message to Slack         | Slack                       | Notify team of new high-priority lead  | 📄 Log Lead to Google Sheet       | -                                     | 📧 Automated Lead Acknowledgment: Sends Slack alert message                                |
| Sticky Note                      | Sticky Note                 | Documentation note                      | -                                | -                                     | 📋 Form Intake & Lead Qualification                                                        |
| Sticky Note1                     | Sticky Note                 | Documentation note                      | -                                | -                                     | 👤 CRM Integration & Priority Handling                                                     |
| Sticky Note2                     | Sticky Note                 | Documentation note                      | -                                | -                                     | 📘/📊 Lead Source Routing & Storage                                                       |
| Sticky Note3                     | Sticky Note                 | Documentation note                      | -                                | -                                     | 📧 Automated Lead Acknowledgment                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Submission Trigger node**  
   - Type: Typeform Trigger  
   - Configure webhook with your Typeform form ID.  
   - Set Typeform API credentials (OAuth or API key).  
   - Position: Start node.

2. **Add If node “💰 Check High-Budget Lead”**  
   - Type: If  
   - Condition: Numeric comparison on `{{$json["What is your Budget"]}} > 5000`  
   - Connect from Typeform Trigger main output.

3. **Add HubSpot node “👤 HubSpot — Create/Update Contact”**  
   - Type: HubSpot (Contact)  
   - Authentication: Use HubSpot App Token credentials.  
   - Parameters:  
     - Email: `{{$json.Email}}`  
     - First Name: `{{$json.Name}}`  
     - Phone Number: `{{$json["Phone Number"]}}`  
     - Custom Property “Budget”: `{{$json["What is your Budget"]}}`  
   - Connect If node’s true output to this node.

4. **Add HubSpot node “📝 HubSpot — Add Priority Task”**  
   - Type: HubSpot (Engagement)  
   - Authentication: Same HubSpot App Token credentials.  
   - Parameters:  
     - Engagement Type: task  
     - Body metadata: `Priority - High (Budget - {{$json["What is your Budget"]}})`  
   - Connect from HubSpot Contact node output.

5. **Add If node “📘 Check if Lead is of (Facebook/SurveyMonkey)”**  
   - Type: If  
   - Conditions:  
     - String contains (case sensitive) `{{$json.lead_source || $json.source}}` contains "Facebook"  
     - OR string equals `{{$json.lead_source || $json.source}}` equals "SurveyMonkey"  
   - Connect HubSpot Add Priority Task node output to this If node.

6. **Add Google Sheets node “📄 Log Lead to Google Sheet”**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Configure spreadsheet ID and sheet name.  
   - Credentials: Google Sheets OAuth2.  
   - Connect If node’s true output to this node.

7. **Add Slack node “😊 Sends Message to Slack”**  
   - Type: Slack  
   - Authentication: Slack API token credentials.  
   - Parameters:  
     - Channel: select your channel ID  
     - Text:  
       ```
       New High Priority Lead:
       Name: {{$json.name}}
       Email: {{$json.email}}
       Number: {{$json.number}}
       Budget: {{$json.budget}}
       Lead of: {{$json.leadof}}
       ```  
   - Connect Google Sheets node output to Slack node.

8. **Set connections:**  
   - Typeform Submission Trigger → Check High-Budget Lead  
   - Check High-Budget Lead (true) → HubSpot Create/Update Contact  
   - HubSpot Create/Update Contact → HubSpot Add Priority Task  
   - HubSpot Add Priority Task → Check if Lead is Facebook/SurveyMonkey  
   - Check if Lead is Facebook/SurveyMonkey (true) → Google Sheets Append  
   - Google Sheets Append → Slack Notification

9. **Credentials setup:**  
   - Typeform API credentials with read webhook access.  
   - HubSpot App Token for CRM and engagement API.  
   - Google Sheets OAuth2 with write access to target sheet.  
   - Slack API token with permission to post messages in target channel.

10. **Test the workflow:**  
    - Submit test entries with various budgets and lead sources.  
    - Verify HubSpot contacts and tasks are created for budgets > 5000.  
    - Check Google Sheets rows for Facebook and SurveyMonkey leads.  
    - Confirm Slack notifications are sent for logged leads.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow uses a budget cutoff of $5,000 to prioritize leads for immediate sales attention.   | Business rule embedded in the "Check High-Budget Lead" node.                                               |
| HubSpot integration requires an App Token with CRM and engagement scopes enabled.                 | HubSpot developer docs: https://developers.hubspot.com/docs/api/app-tokens                                 |
| Google Sheets logging supports marketing analysis and campaign tracking by source.               | Google Sheets API docs: https://developers.google.com/sheets/api                                          |
| Slack notifications provide real-time alerts to sales or marketing teams for quick lead follow-up.| Slack API docs: https://api.slack.com/messaging/webhooks                                                    |
| Sticky notes throughout the workflow clarify functional blocks and improve maintainability.      | Sticky notes visually document the logic flow and use case rationale within the n8n editor workspace.      |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow designed with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---