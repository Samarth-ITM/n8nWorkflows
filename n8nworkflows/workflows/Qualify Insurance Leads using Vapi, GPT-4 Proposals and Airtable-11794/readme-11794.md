Qualify Insurance Leads using Vapi, GPT-4 Proposals and Airtable

https://n8nworkflows.xyz/workflows/qualify-insurance-leads-using-vapi--gpt-4-proposals-and-airtable-11794


# Qualify Insurance Leads using Vapi, GPT-4 Proposals and Airtable

### 1. Workflow Overview

This workflow automates the qualification and follow-up process for insurance leads by integrating form submissions, AI voice calls, call analysis, AI-generated insurance proposals, and CRM updates. It targets insurance brokers who want to accelerate lead engagement and qualification by combining Vapi AI voice calls, GPT-4 proposal generation, Airtable CRM, Gmail emailing, and Slack notifications.

The workflow is organized into these logical blocks:

- **1.1 Lead Ingestion & Voice Trigger:** Captures lead data from a submitted form, logs it in Airtable, and triggers an AI voice outbound call via Vapi.
- **1.2 Call Processing & Qualification Logic:** Receives call summary and metadata from Vapi via webhook, extracts relevant data, and evaluates if the lead is qualified based on call content.
- **1.3 Qualified Lead Actions:** Updates Airtable with qualified status, generates a custom insurance proposal using GPT-4, and emails the blueprint to the lead.
- **1.4 Unqualified Lead Actions:** Updates Airtable with unqualified status and sends a Slack notification to alert the sales team with details.
- **1.5 Error Handling:** Monitors for email send failures and alerts the team on Slack with the generated proposal text for manual follow-up.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Ingestion & Voice Trigger

**Overview:**  
This block captures leads submitted via a web form, creates a new record in Airtable to log their data, and immediately triggers an AI voice call to the lead using Vapi to gather more information.

**Nodes Involved:**  
- On form submission  
- Create a record  
- Vapi call  
- Sticky Note (Lead Ingestion & Voice Trigger)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point for new leads via a web form titled "Harvex Insurance".  
  - Config: Collects "name" (text), "Email" (email), and "Phone Number" (number).  
  - Outputs raw form data on submission.  
  - Edge Cases: Form validation issues, missing fields, or webhook failures.

- **Create a record**  
  - Type: Airtable node  
  - Role: Creates a new lead record in Airtable base "Leads" in "Table 1".  
  - Config: Maps form fields "name", "Email", and "Phone Number" (converted to string) to Airtable columns.  
  - Credentials: Airtable Personal Access Token.  
  - Input: Data from form submission.  
  - Output: Airtable record metadata.  
  - Edge Cases: Airtable API limits, authentication failures.

- **Vapi call**  
  - Type: HTTP Request  
  - Role: Initiates an AI voice call via Vapi API to the lead’s phone number with assistant and phone number IDs.  
  - Config: POST request with JSON body containing assistantId, phoneNumberId, and customer details (name, phone with country code '+230', email).  
  - Auth: HTTP Header Auth with Authorization header.  
  - Input: Airtable record data (phone number, name, email).  
  - Edge Cases: API errors, invalid phone numbers, authentication failure, or network timeouts.

- **Sticky Note**  
  - Content: Explains this block captures form leads, logs to Airtable, and triggers the voice call.

---

#### 1.2 Call Processing & Qualification Logic

**Overview:**  
Receives call results from Vapi via webhook, cleans and extracts relevant fields such as call summary, success status, and type of insurance. Determines if the lead is qualified based on AI evaluation and extracted insurance type.

**Nodes Involved:**  
- Post call (Webhook)  
- Clean data  
- Is Qualified ? (If node)  
- Sticky Note (Call Processing & Logic)

**Node Details:**

- **Post call**  
  - Type: Webhook  
  - Role: Receives POST requests from Vapi containing call transcript and analysis after call finishes.  
  - Config: Path set uniquely, listens for POST, responds with last node output.  
  - Edge Cases: Unauthorized calls, malformed payloads, webhook downtime.

- **Clean data**  
  - Type: Set node  
  - Role: Extracts and renames key fields from webhook JSON: call summary, success evaluation, date, time, email, name, phone number, and insurance type.  
  - Key Expressions: Uses expressions to parse nested JSON fields from Vapi's response body.  
  - Output: Structured JSON with simplified fields for downstream logic.  
  - Edge Cases: Missing or malformed data, expression evaluation failures.

- **Is Qualified ?**  
  - Type: If node  
  - Role: Decision point checking if call analysis was successful (Success = true) and if "Type of Insurance" is not empty.  
  - Condition: Using Regex and non-empty string checks on extracted fields.  
  - Outputs: True branch for qualified leads, False branch for unqualified.  
  - Edge Cases: Logic errors, unexpected data formats.

- **Sticky Note**  
  - Content: Describes reception of call summary, entity extraction, and qualification decision logic.

---

#### 1.3 Qualified Lead Actions

**Overview:**  
For leads determined qualified, this block updates Airtable to record the qualified status and call summary, uses GPT-4 to generate a professional insurance blueprint email, and emails the proposal to the lead.

**Nodes Involved:**  
- Update record1 (Airtable update)  
- Prepare Blueprint (OpenAI GPT-4 node)  
- Send a message (Gmail email)  
- Error Handling (Slack alert on email failure)  
- Sticky Note (Qualified Lead Actions)

**Node Details:**

- **Update record1**  
  - Type: Airtable node (update)  
  - Role: Updates existing lead record by matching on Email, setting Status to "Qualify", updating Call summary and Type of Insurance.  
  - Input: Data from "Is Qualified ?" node true branch.  
  - Credentials: Airtable Personal Access Token.  
  - Edge Cases: Record not found, API failures.

- **Prepare Blueprint**  
  - Type: OpenAI (Langchain) node  
  - Role: Generates a customized insurance proposal email using GPT-4.1-mini model.  
  - Config: System prompt styles the AI as "Alex Mercer", includes detailed instructions on email structure, markdown table for coverage, and tone.  
  - Inputs: Fields such as client name, insurance type, and call summary from Airtable update node.  
  - Credentials: OpenAI API.  
  - Edge Cases: API rate limits, prompt syntax errors, model unavailability.

- **Send a message (Gmail)**  
  - Type: Gmail node  
  - Role: Sends the generated proposal email to the qualified lead’s email address.  
  - Config: Text email, subject "Your insurance blueprint", message body from GPT output.  
  - Credentials: Gmail OAuth2.  
  - On error: Continues flow to Error Handling node.  
  - Edge Cases: Email delivery failure, OAuth token expiration.

- **Error Handling**  
  - Type: Slack node  
  - Role: Sends a critical alert to Slack channel if email sending fails, including the blueprint text for manual sending.  
  - Config: Message includes user name, email, and blueprint text extracted from previous nodes.  
  - Credentials: Slack OAuth2.  
  - Edge Cases: Slack API failures, missing context data.

- **Sticky Note**  
  - Content: Explains updating CRM, generating blueprint with GPT-4, and emailing client.

---

#### 1.4 Unqualified Lead Actions

**Overview:**  
Handles leads that fail qualification by updating Airtable status to "Unqualified" and sending a Slack notification to the sales team with disqualification reasons.

**Nodes Involved:**  
- Update record (Airtable update)  
- Send a message1 (Slack notification)  
- Sticky Note (Unqualified Lead Actions)

**Node Details:**

- **Update record**  
  - Type: Airtable node (update)  
  - Role: Updates lead record matching on Email, setting Status to "Unqualified" and storing call summary and phone number.  
  - Credentials: Airtable Personal Access Token.  
  - Edge Cases: Record not found, API errors.

- **Send a message1**  
  - Type: Slack node  
  - Role: Posts an alert message in Slack channel "all-harvex" notifying team of unqualified lead with name, phone, call summary, and reason.  
  - Config: Uses OAuth2 authentication.  
  - Edge Cases: Slack API errors, authentication failure.

- **Sticky Note**  
  - Content: Describes CRM update and Slack alert for unqualified leads.

---

#### 1.5 Error Handling

**Overview:**  
Dedicated Slack notification node alerts the team if the email sending node fails, ensuring manual follow-up can be done.

**Nodes Involved:**  
- Error Handling (Slack alert)  
- Sticky Note (Error Handling)

**Node Details:**

- **Error Handling** (covered above in 1.3)  
  - On email send failure, posts critical error alert with relevant user data and blueprint text.

- **Sticky Note**  
  - Content: Notes that failure in email node triggers Slack alert for manual intervention.

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                                 | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                             |
|--------------------|-------------------------|------------------------------------------------|-----------------------|----------------------------|-------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger            | Entry point; captures lead form submission     | -                     | Create a record             |                                                                                                       |
| Create a record     | Airtable                | Logs new lead data in Airtable                  | On form submission    | Vapi call                  |                                                                                                       |
| Vapi call          | HTTP Request            | Triggers AI voice call to lead via Vapi        | Create a record       |                            |                                                                                                       |
| Post call          | Webhook                 | Receives call summary & analysis from Vapi     | -                     | Clean data                 |                                                                                                       |
| Clean data         | Set                     | Extracts relevant call info and metadata        | Post call             | Is Qualified ?             |                                                                                                       |
| Is Qualified ?     | If                      | Decision node: determines lead qualification    | Clean data             | Update record1, Update record |                                                                                                       |
| Update record1     | Airtable                | Updates Airtable to mark lead as qualified      | Is Qualified ? (true)  | Prepare Blueprint           | Qualified Lead Actions: CRM update, generate blueprint, email client                                  |
| Prepare Blueprint  | OpenAI (Langchain)      | Generates insurance proposal email via GPT-4    | Update record1         | Send a message              |                                                                                                       |
| Send a message     | Gmail                   | Emails the generated proposal to qualified lead | Prepare Blueprint      | Error Handling (on failure) |                                                                                                       |
| Error Handling     | Slack                   | Alerts team on email failure for manual follow-up | Send a message (error) |                            | Error Handling: Alerts team via Slack if email fails                                                 |
| Update record      | Airtable                | Updates Airtable to mark lead as unqualified    | Is Qualified ? (false) | Send a message1             | Unqualified Lead Actions: CRM update and Slack alert                                                 |
| Send a message1    | Slack                   | Notifies sales team about unqualified lead      | Update record          |                            |                                                                                                       |
| Sticky Note        | Sticky Note             | Lead Ingestion & Voice Trigger description       | -                     | -                          | ## Lead Ingestion & Voice Trigger Captures the lead from the form submitted then logs the data  in Airtable followed by the phone call via Vapi. |
| Sticky Note1       | Sticky Note             | Call Processing & Logic description               | -                     | -                          | ## Call Processing & Logic Receives the call summary and entity data from Vapi. It determines if the lead provided enough information to be considered as  "Qualified." |
| Sticky Note2       | Sticky Note             | Workflow purpose and overview                     | -                     | -                          | ## Qualify Insurance Leads using Vapi, GPT-4 Proposals and Airtable... [full description in node]    |
| Sticky Note3       | Sticky Note             | Qualified Lead Actions description                | -                     | -                          | ## Qualified Lead Actions If the lead is  qualified,  the CRM is updated by qualifying the person and  GPT-4 is used to draft a custom insurance blueprint. This blueprint is sent to the qualified leads via email |
| Sticky Note4       | Sticky Note             | Unqualified Lead Actions description              | -                     | -                          | ## Unqualified Lead Actions If the lead is unqualified the CRM updates and alerts the team via Slack with the reason for disqualification. |
| Sticky Note5       | Sticky Note             | Error Handling description                         | -                     | -                          | ## Error Handling If the email fails to send, this alerts the team via Slack with the generated proposal text so it can be sent manually. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form title as "Harvex Insurance"  
   - Add form fields:  
     - Text field "name" with placeholder "Type name"  
     - Email field "Email" with placeholder "Type your Email"  
     - Number field "Phone Number" with placeholder "Type your phone number"

2. **Create Airtable Node for Record Creation**  
   - Type: Airtable  
   - Name: "Create a record"  
   - Operation: Create  
   - Configure Airtable credentials with Personal Access Token  
   - Select base "Leads" and table "Table 1"  
   - Map fields:  
     - name -> {{$json.name}}  
     - Email -> {{$json.Email}}  
     - Phone number -> {{$json["Phone Number"].toString()}}  
   - Connect output of "On form submission" to this node.

3. **Create HTTP Request Node for Vapi Call**  
   - Type: HTTP Request  
   - Name: "Vapi call"  
   - Method: POST  
   - URL: https://api.vapi.ai/call/phone  
   - Authentication: HTTP Header Auth with your Authorization header credential  
   - Body (JSON):  
     ```json
     {
       "assistantId": "ENTER_ASSISTANT_ID_HERE",
       "phoneNumberId": "ENTER_PHONE_NUMBER_ID_HERE",
       "customer": {
         "number": "{{ '+230' + $json.fields['Phone number'] }}",
         "name": "{{ $json.fields.name }}"
       },
       "assistantOverrides": {
         "variableValues": {
           "name": "{{ $json.fields.name }}",
           "email": "{{ $json.fields.Email }}"
         },
         "server": {
           "url": ""
         },
         "serverMessages": [
           "end-of-call-report"
         ]
       }
     }
     ```
   - Connect output of "Create a record" to this node.

4. **Create Webhook Node for Vapi Post Call**  
   - Type: Webhook  
   - Name: "Post call"  
   - HTTP Method: POST  
   - Set a unique path (e.g., "cf226daa-a404-4c24-a44a-33526891e5f2")  
   - This node receives call summaries from Vapi.

5. **Create Set Node to Clean Data**  
   - Type: Set  
   - Name: "Clean data"  
   - Assign variables extracting from webhook JSON:  
     - Call summary = {{$json.body.message.analysis.summary}}  
     - Success = {{$json.body.message.analysis.successEvaluation}}  
     - date = {{$json.body.message.artifact.variableValues.date}}  
     - Time = {{$json.body.message.artifact.variableValues.time}}  
     - Email = {{$json.body.message.artifact.variableValues.email}}  
     - Name = {{$json.body.message.assistant.variableValues.name}}  
     - phone number = {{$json.body.message.call.customer.number}}  
     - Type of Insurance = {{$json.body.message.artifact.messages[4].message}}  
   - Connect output of "Post call" to this node.

6. **Create If Node to Determine Qualification**  
   - Type: If  
   - Name: "Is Qualified ?"  
   - Conditions (AND):  
     - Success matches regex "true" (string)  
     - Type of Insurance is not empty  
   - Connect output of "Clean data" to this node.

7. **Create Airtable Update Node for Qualified Leads**  
   - Type: Airtable  
   - Name: "Update record1"  
   - Operation: Update  
   - Credentials: Airtable Personal Access Token  
   - Base: "Leads", Table: "Table 1"  
   - Match by: Email  
   - Fields to update:  
     - Status: "Qualify"  
     - Call summary: {{$json['Call summary ']}}  
     - Type of Insurance: {{$json['Type of Insurance']}}  
   - Connect True output of "Is Qualified ?" to this node.

8. **Create OpenAI Node to Prepare Insurance Blueprint**  
   - Type: OpenAI (Langchain)  
   - Name: "Prepare Blueprint"  
   - Model: "gpt-4.1-mini"  
   - System Prompt: [Use the detailed system prompt text from the workflow that directs the AI to generate a professional insurance blueprint email including markdown tables and bullet points, tailored using lead data.]  
   - Credentials: OpenAI API Key  
   - Connect output of "Update record1" to this node.

9. **Create Gmail Node to Send Email**  
   - Type: Gmail  
   - Name: "Send a message"  
   - Send To: {{$node["Update record1"].json.Email}}  
   - Subject: "Your insurance blueprint"  
   - Message: {{$json.output[0].content[0].text}}  
   - Credentials: Gmail OAuth2  
   - Connect output of "Prepare Blueprint" to this node.  
   - On error, continue to next node.

10. **Create Slack Node for Email Failure Handling**  
    - Type: Slack  
    - Name: "Error Handling"  
    - Message: Critical alert including user name, email, and blueprint text from previous nodes.  
    - Channel: Slack channel ID (e.g., "all-harvex")  
    - Credentials: Slack OAuth2  
    - Connect Error output of "Send a message" to this node.

11. **Create Airtable Update Node for Unqualified Leads**  
    - Type: Airtable  
    - Name: "Update record"  
    - Operation: Update  
    - Credentials: Airtable Personal Access Token  
    - Base: "Leads", Table: "Table 1"  
    - Match by: Email  
    - Fields to update:  
      - Status: "Unqualified"  
      - Call summary: {{$json['Call summary ']}}  
      - Phone number: {{$json['phone number']}}  
    - Connect False output of "Is Qualified ?" to this node.

12. **Create Slack Node to Notify Team of Unqualified Leads**  
    - Type: Slack  
    - Name: "Send a message1"  
    - Text: Unqualified Lead Alert with fields Name, Phone, Call summary, and Reason.  
    - Channel: Slack channel ID (e.g., "all-harvex")  
    - Credentials: Slack OAuth2  
    - Connect output of "Update record" to this node.

13. **Add Sticky Notes**  
    - Add descriptive sticky notes to each logical block for documentation and future reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow automates the "speed-to-lead" process for insurance brokers by integrating AI voice calls (Vapi), AI-generated proposals (OpenAI GPT-4), Airtable CRM, Gmail, and Slack for notifications.                                                                                                                                                                                | Workflow description in Sticky Note2                                                                                    |
| OpenAI system prompt is highly customized to produce professional, concise insurance proposal emails with markdown tables and bullet points, ensuring clarity and professionalism.                                                                                                                                                                                                     | See "Prepare Blueprint" node system prompt                                                                               |
| Vapi API requires valid Assistant ID and Phone Number ID; these must be configured in the HTTP Request node for outbound calls.                                                                                                                                                                                                                                                        | Vapi call node configuration                                                                                            |
| Airtable base "Leads" and table "Table 1" must have columns for name, Email, Phone number, Status, Call summary, and Type of Insurance with appropriate data types and permission settings.                                                                                                                                                                                             | Airtable nodes configuration                                                                                            |
| Gmail OAuth2 credentials must allow sending emails on behalf of the sender; ensure token refresh mechanisms are in place.                                                                                                                                                                                                                                                                 | Gmail node details                                                                                                      |
| Slack OAuth2 credentials must have permission to post messages to designated channels for team notifications and error alerts.                                                                                                                                                                                                                                                          | Slack nodes configuration                                                                                               |
| The workflow assumes country code +230 for phone numbers; adjust as needed for your region.                                                                                                                                                                                                                                                                                              | Vapi call phone number concatenation                                                                                   |
| For production, validate all external API error handling and consider adding retries or alerts for failed Airtable or Vapi calls for robustness.                                                                                                                                                                                                                                         | General best practices                                                                                                  |
| Workflow is inactive by default; activate after proper credential and node configuration.                                                                                                                                                                                                                                                                                                | n8n workflow settings                                                                                                   |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.