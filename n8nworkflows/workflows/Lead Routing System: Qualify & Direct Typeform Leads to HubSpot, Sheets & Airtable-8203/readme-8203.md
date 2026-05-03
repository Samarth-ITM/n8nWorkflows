Lead Routing System: Qualify & Direct Typeform Leads to HubSpot, Sheets & Airtable

https://n8nworkflows.xyz/workflows/lead-routing-system--qualify---direct-typeform-leads-to-hubspot--sheets---airtable-8203


# Lead Routing System: Qualify & Direct Typeform Leads to HubSpot, Sheets & Airtable

### 1. Workflow Overview

This workflow automates lead routing and qualification for multi-source leads collected primarily via Typeform. It is designed to:

- Capture and qualify incoming leads based on budget,
- Update CRM records in HubSpot and prioritize high-budget leads,
- Route leads based on their source (Facebook or SurveyMonkey) to appropriate storage systems (Google Sheets or Airtable),
- Send an automated acknowledgment email to leads confirming receipt.

The workflow consists of four main logical blocks:

- **1.1 Form Intake & Lead Qualification**: Triggered by new Typeform submissions, it checks if the lead’s budget exceeds a high-priority threshold.
- **1.2 CRM Integration & Priority Handling**: Creates or updates contacts in HubSpot, adds priority tasks for high-budget leads.
- **1.3 Lead Source Routing & Storage**: Routes leads to Google Sheets if from Facebook, or Airtable if from SurveyMonkey.
- **1.4 Automated Lead Acknowledgment**: Sends a Gmail auto-response to all routed leads after storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Intake & Lead Qualification

**Overview:**  
This block captures new lead data from Typeform submissions and evaluates if the lead qualifies as high priority based on budget (> $5,000). It initiates downstream routing based on this qualification.

**Nodes Involved:**  
- 📋 Typeform Submission Trigger  
- 💰 Check High-Budget Lead

**Node Details:**

- **📋 Typeform Submission Trigger**  
  - *Type:* Typeform Trigger  
  - *Role:* Receives lead data from Typeform form submissions in real-time via webhook.  
  - *Configuration:* Linked to a specific Typeform form ID; uses stored Typeform API credentials for authentication.  
  - *Key Variables:* Entire submission JSON, including "What is your Budget".  
  - *Inputs:* External webhook from Typeform.  
  - *Outputs:* Passes submission data to budget check node.  
  - *Edge Cases:* Webhook failure, credential expiration, malformed payloads.

- **💰 Check High-Budget Lead**  
  - *Type:* If Node (conditional logic)  
  - *Role:* Determines if the submitted budget exceeds 5,000 USD.  
  - *Configuration:* Checks if `{{ $json['What is your Budget'] }}` > 5000.  
  - *Key Expressions:* Number comparison on budget field.  
  - *Input:* From Typeform Trigger node.  
  - *Output:* If true, routes to HubSpot contact creation; if false, no direct output (workflow ends or could be extended).  
  - *Edge Cases:* Missing or non-numeric budget fields, case sensitivity handled by strict validation.

---

#### 2.2 CRM Integration & Priority Handling

**Overview:**  
For leads flagged as high-budget, this block creates or updates the contact in HubSpot CRM and logs a priority task to prompt timely sales follow-up.

**Nodes Involved:**  
- 👤 HubSpot — Create/Update Contact  
- 📝 HubSpot — Add Priority Task

**Node Details:**

- **👤 HubSpot — Create/Update Contact**  
  - *Type:* HubSpot Node (CRM Contact Management)  
  - *Role:* Creates or updates HubSpot contact records with lead information.  
  - *Configuration:* Uses email as unique identifier; updates first name, phone number, and custom property “Budget” from lead data.  
  - *Credentials:* Uses HubSpot App Token for authentication.  
  - *Input:* True output from budget check node.  
  - *Output:* Passes contact data to task creation node.  
  - *Edge Cases:* Authentication failure, API rate limits, missing email field.

- **📝 HubSpot — Add Priority Task**  
  - *Type:* HubSpot Node (Engagement Creation)  
  - *Role:* Creates a task engagement in HubSpot linked to the contact, with a message indicating high priority and budget info.  
  - *Configuration:* Task type engagement, body message includes budget value dynamically.  
  - *Credentials:* Same HubSpot App Token.  
  - *Input:* From contact creation node.  
  - *Output:* Continues to lead source routing (Facebook lead check).  
  - *Edge Cases:* Task creation failure, API limits.

---

#### 2.3 Lead Source Routing & Storage

**Overview:**  
This block routes leads according to their source platform, logging Facebook leads to Google Sheets and SurveyMonkey leads to Airtable for respective marketing and campaign management.

**Nodes Involved:**  
- 📘 Check if Facebook Lead  
- 📊 Check if SurveyMonkey Lead  
- 📄 Log Facebook Lead to Google Sheet  
- 📦 Store SurveyMonkey Lead in Airtable

**Node Details:**

- **📘 Check if Facebook Lead**  
  - *Type:* If Node  
  - *Role:* Checks if the lead source contains "Facebook".  
  - *Expression:* `{{$json.lead_source || $json.source}}` contains "Facebook".  
  - *Input:* From HubSpot priority task node.  
  - *Outputs:* True → Facebook logging node; False → SurveyMonkey check node.  
  - *Edge Cases:* Missing or malformed source fields.

- **📊 Check if SurveyMonkey Lead**  
  - *Type:* If Node  
  - *Role:* Checks if lead source contains "SurveyMonkey".  
  - *Expression:* Similar logic as Facebook check but for "SurveyMonkey".  
  - *Input:* False output from Facebook check node or from Typeform trigger if extended.  
  - *Outputs:* True → Airtable storage node; False → Sends email response directly.  
  - *Edge Cases:* Same as above.

- **📄 Log Facebook Lead to Google Sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends Facebook lead data into a designated sheet for marketing analysis.  
  - *Configuration:* Uses OAuth2 credentials; specifies document and sheet by ID; appends rows with mapped lead data.  
  - *Input:* True path from Facebook lead check.  
  - *Output:* Leads to Gmail auto-response node.  
  - *Edge Cases:* API quota exceeded, invalid sheet IDs, credential expiration.

- **📦 Store SurveyMonkey Lead in Airtable**  
  - *Type:* Airtable Node  
  - *Role:* Creates new records in Airtable base and table to store SurveyMonkey leads.  
  - *Configuration:* Uses OAuth2 credentials; maps fields like Name, Email, Phone, Message.  
  - *Input:* True path from SurveyMonkey lead check.  
  - *Output:* Leads to Gmail auto-response node.  
  - *Edge Cases:* Airtable API limits, base/table misconfiguration, missing required fields.

---

#### 2.4 Automated Lead Acknowledgment

**Overview:**  
Sends a Gmail auto-response confirming receipt of the inquiry and setting expectations for follow-up within 24 hours. This happens after lead data is stored in either Google Sheets or Airtable.

**Nodes Involved:**  
- 📧 Send Gmail Auto-Response

**Node Details:**

- **📧 Send Gmail Auto-Response**  
  - *Type:* Gmail Node (Send Email)  
  - *Role:* Sends a templated thank-you email to the lead’s email address captured earlier.  
  - *Configuration:* Uses OAuth2 Gmail credentials; email message personalizes greeting with lead’s first name or full name; subject and body are predefined.  
  - *Input:* From both Google Sheets and Airtable nodes (parallel paths converge here).  
  - *Output:* Workflow end.  
  - *Edge Cases:* Missing or invalid email, Gmail API limits, credential expiration.

---

### 3. Summary Table

| Node Name                        | Node Type            | Functional Role                              | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                             |
|---------------------------------|----------------------|----------------------------------------------|------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| 📋 Typeform Submission Trigger   | Typeform Trigger     | Capture new leads from Typeform submissions | —                            | 💰 Check High-Budget Lead        | 📋 Form Intake & Lead Qualification: Captures leads and evaluates budget for priority routing.                          |
| 💰 Check High-Budget Lead         | If Node              | Qualify leads by budget > $5,000              | 📋 Typeform Submission Trigger | 👤 HubSpot — Create/Update Contact | Same as above                                                                                                           |
| 👤 HubSpot — Create/Update Contact| HubSpot Node         | Create/update contact in HubSpot CRM          | 💰 Check High-Budget Lead      | 📝 HubSpot — Add Priority Task    | 👤 CRM Integration & Priority Handling: Updates HubSpot contact and logs priority task for high-budget leads.           |
| 📝 HubSpot — Add Priority Task    | HubSpot Node         | Add priority follow-up task in HubSpot       | 👤 HubSpot — Create/Update Contact | 📘 Check if Facebook Lead       | Same as above                                                                                                           |
| 📘 Check if Facebook Lead         | If Node              | Check if lead source is Facebook              | 📝 HubSpot — Add Priority Task  | 📄 Log Facebook Lead to Google Sheet, 📊 Check if SurveyMonkey Lead | 📘/📊 Lead Source Routing & Storage: Routes Facebook leads to Google Sheets, SurveyMonkey leads to Airtable.            |
| 📊 Check if SurveyMonkey Lead     | If Node              | Check if lead source is SurveyMonkey          | 📘 Check if Facebook Lead       | 📦 Store SurveyMonkey Lead in Airtable, 📧 Send Gmail Auto-Response | Same as above                                                                                                           |
| 📄 Log Facebook Lead to Google Sheet | Google Sheets Node | Append Facebook leads to Google Sheet         | 📘 Check if Facebook Lead       | 📧 Send Gmail Auto-Response       | 📧 Automated Lead Acknowledgment: Sends confirmation email after logging lead data.                                     |
| 📦 Store SurveyMonkey Lead in Airtable | Airtable Node     | Store SurveyMonkey leads in Airtable          | 📊 Check if SurveyMonkey Lead   | 📧 Send Gmail Auto-Response       | Same as above                                                                                                           |
| 📧 Send Gmail Auto-Response       | Gmail Node           | Send automated acknowledgment email           | 📄 Log Facebook Lead to Google Sheet, 📦 Store SurveyMonkey Lead in Airtable | —                               | Same as above                                                                                                           |
| Sticky Note                      | Sticky Note          | Documentation block for Form Intake & Lead Qualification | —                            | —                               | ## 📋 Form Intake & Lead Qualification: Describes initial lead capture and budget checking process.                     |
| Sticky Note1                     | Sticky Note          | Documentation block for CRM Integration & Priority Handling  | —                            | —                               | ## 👤 CRM Integration & Priority Handling: Details HubSpot contact update and task creation.                             |
| Sticky Note2                     | Sticky Note          | Documentation block for Lead Source Routing & Storage          | —                            | —                               | ## 📘/📊 Lead Source Routing & Storage: Explains routing leads to Google Sheets or Airtable based on source.             |
| Sticky Note3                     | Sticky Note          | Documentation block for Automated Lead Acknowledgment          | —                            | —                               | ## 📧 Automated Lead Acknowledgment: Covers sending Gmail auto-response after data logging.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it appropriately (e.g., "Multi-Source Lead Routing").**

2. **Add the node: Typeform Submission Trigger**  
   - Node Type: Typeform Trigger  
   - Configure: Set the "Form ID" to your Typeform form identifier.  
   - Credentials: Select or create a Typeform API credential with appropriate scopes.  
   - Save.

3. **Add the node: Check High-Budget Lead (If Node)**  
   - Node Type: If  
   - Condition: Set to check if `{{$json['What is your Budget']}}` (number) is greater than 5000.  
   - Connect the Typeform Trigger node output to this node input.

4. **Add the node: HubSpot — Create/Update Contact**  
   - Node Type: HubSpot  
   - Operation: Create or update contact.  
   - Parameters:  
     - Email: `{{$json.Email}}`  
     - Additional Fields:  
       - First Name: `{{$json.Name}}`  
       - Phone Number: `{{$json['Phone Number']}}`  
       - Custom Property "Budget": `{{$json['What is your Budget']}}`  
   - Credentials: Select HubSpot App Token credential.  
   - Connect the "true" output of the budget check node to this node.

5. **Add the node: HubSpot — Add Priority Task**  
   - Node Type: HubSpot  
   - Operation: Create engagement of type "task".  
   - Parameters:  
     - Body: `Priority - High (Budget - {{$json['What is your Budget']}})`  
   - Credentials: Same HubSpot App Token credential as above.  
   - Connect output of HubSpot contact node to this node.

6. **Add the node: Check if Facebook Lead (If Node)**  
   - Node Type: If  
   - Condition: Check if `{{$json.lead_source || $json.source}}` contains "Facebook" (string contains).  
   - Connect output of HubSpot task node to this node.

7. **Add the node: Check if SurveyMonkey Lead (If Node)**  
   - Node Type: If  
   - Condition: Check if `{{$json.lead_source || $json.source}}` contains "SurveyMonkey".  
   - Connect the "false" output of Facebook check node to this node.

8. **Add the node: Log Facebook Lead to Google Sheet**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Configure:  
     - Document ID: Your Google Sheets document ID  
     - Sheet Name: Your sheet name or ID  
     - Map columns as needed (e.g., Name, Email, etc.)  
   - Credentials: Use Google Sheets OAuth2 credential.  
   - Connect the "true" output of Facebook check node here.

9. **Add the node: Store SurveyMonkey Lead in Airtable**  
   - Node Type: Airtable  
   - Operation: Create record  
   - Configure:  
     - Base ID and Table name/ID for your Airtable setup  
     - Map fields such as Name, Email, Phone Number, Message  
   - Credentials: Use Airtable OAuth2 credential.  
   - Connect the "true" output of SurveyMonkey check node here.

10. **Add the node: Send Gmail Auto-Response**  
    - Node Type: Gmail  
    - Operation: Send Email  
    - Configure:  
      - To: `{{$json.email}}`  
      - Subject: "Thanks for reaching out — We'll get back soon"  
      - Message:  
        ```
        Hi {{$json.name || $json.first_name}},

        Thank you for your interest. Our sales team will reach out within 24 hours.

        Best regards,
        Sales Team
        ```  
    - Credentials: Use Gmail OAuth2 credential.  
    - Connect both Google Sheets node and Airtable node outputs to this Gmail node.

11. **Finalize connections:**  
    - Connect "true" output of Check High-Budget Lead node to HubSpot contact node.  
    - Connect HubSpot contact node to HubSpot task node.  
    - Connect HubSpot task node to Facebook lead check node.  
    - Connect Facebook check node "true" to Google Sheets node.  
    - Connect Facebook check node "false" to SurveyMonkey check node.  
    - Connect SurveyMonkey check node "true" to Airtable node.  
    - Connect SurveyMonkey check node "false" (if applicable) to Gmail node or end workflow.  
    - Connect Google Sheets node output to Gmail node.  
    - Connect Airtable node output to Gmail node.

12. **Test workflow with sample leads from Typeform to verify branching, data insertion, and email sending.**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow uses multi-platform integration: Typeform, HubSpot, Google Sheets, Airtable, Gmail.                          | Demonstrates cross-system lead routing and CRM synchronization.                                                 |
| HubSpot API usage requires an App Token with permissions to manage contacts and engagements (tasks).                  | HubSpot developer docs: https://developers.hubspot.com/docs/api/overview                                          |
| Google Sheets and Airtable nodes use OAuth2 authentication to ensure secure API access.                               | Google Sheets API: https://developers.google.com/sheets/api; Airtable API: https://airtable.com/api               |
| Gmail auto-response email template sets customer expectations for 24-hour sales follow-up.                           | Email personalization uses expressions to include lead name if available.                                        |
| Design recommendation: Validate incoming data fields for presence and format to reduce errors (e.g., budget numeric). | May add additional error handling nodes or notifications for failed API calls.                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.