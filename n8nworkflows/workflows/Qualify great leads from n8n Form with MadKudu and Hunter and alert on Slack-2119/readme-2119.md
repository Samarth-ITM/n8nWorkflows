Qualify great leads from n8n Form with MadKudu and Hunter and alert on Slack

https://n8nworkflows.xyz/workflows/qualify-great-leads-from-n8n-form-with-madkudu-and-hunter-and-alert-on-slack-2119


# Qualify great leads from n8n Form with MadKudu and Hunter and alert on Slack

### 1. Workflow Overview

This n8n workflow is designed to qualify leads submitted through a web form by validating their email addresses, scoring their potential using MadKudu, and alerting a Slack channel if the lead meets a defined quality threshold. It is primarily targeted at businesses who want to automate lead qualification and prioritize outreach based on lead scoring.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures lead information via an n8n embedded form trigger.
- **1.2 Email Validation:** Uses Hunter.io to verify if the submitted email is valid.
- **1.3 Lead Scoring:** Sends the validated email to MadKudu to retrieve a customer fit score.
- **1.4 Qualification Decision:** Checks if the lead’s customer fit score exceeds a threshold (60).
- **1.5 Notification or No-op:** Sends a formatted Slack message for qualified leads or does nothing otherwise.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new lead submissions through an n8n Form Trigger node. It collects the business email and initiates the workflow.

- **Nodes Involved:**  
  - n8n Form Trigger

- **Node Details:**  
  - **Name:** n8n Form Trigger  
  - **Type:** Trigger (formTrigger)  
  - **Configuration:**  
    - Webhook path: unique identifier `0bf8840f-1cc4-46a9-86af-a3fa8da80608`  
    - Form title: "Contact us"  
    - Form fields: Single field requesting “What’s your business email?”  
    - Form description: "We'll get back to you soon"  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connects to "Verify email with Hunter" node  
  - **Edge Cases / Failures:**  
    - Network issues on webhook reception  
    - Malformed form data or missing email input  
  - **Version Requirements:** n8n supporting webhook Form Trigger (typeVersion 2)  

---

#### 2.2 Email Validation

- **Overview:**  
  Validates the submitted email address using Hunter.io’s email verifier to ensure it is deliverable before proceeding.

- **Nodes Involved:**  
  - Verify email with Hunter  
  - Check if the email is valid  
  - Email is not valid, do nothing

- **Node Details:**  

  - **Verify email with Hunter**  
    - Type: Hunter node (email verifier operation)  
    - Configuration:  
      - Email input: taken from form field `"What's your business email?"`  
    - Credentials: Hunter API key  
    - Input: from n8n Form Trigger  
    - Output: to "Check if the email is valid" (If node)  
    - Failures: API key issues, rate limits, invalid email formats  

  - **Check if the email is valid**  
    - Type: If node  
    - Condition: Checks if `status` field equals `"valid"` (case-sensitive, strict) in Hunter API response  
    - Input: from "Verify email with Hunter"  
    - Outputs:  
      - TRUE: proceeds to "Score lead with MadKudu"  
      - FALSE: proceeds to "Email is not valid, do nothing"  
    - Edge cases: Missing or unexpected response fields may cause expression errors  

  - **Email is not valid, do nothing**  
    - Type: No Operation (NoOp) node  
    - Role: Silently ends the workflow branch for invalid emails without error  
    - Input: from FALSE output of "Check if the email is valid"  

---

#### 2.3 Lead Scoring

- **Overview:**  
  For emails validated as deliverable, this block queries MadKudu’s API to score the lead’s customer fit.

- **Nodes Involved:**  
  - Score lead with MadKudu  
  - if customer fit score > 60

- **Node Details:**  

  - **Score lead with MadKudu**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL: `https://api.madkudu.com/v1/persons?email={{ $json.email }}` (dynamically set from input email)  
      - Auth: HTTP header authentication with MadKudu API token  
    - Input: TRUE output from "Check if the email is valid"  
    - Output: to "if customer fit score > 60"  
    - Edge cases: API downtime, invalid or expired credentials, response format changes  
    - Version: HTTP Request node v4.1  

  - **if customer fit score > 60**  
    - Type: If node  
    - Condition: Checks if `properties.customer_fit.score` is greater than 60 (numeric comparison)  
    - Input: from "Score lead with MadKudu"  
    - Outputs:  
      - TRUE: proceeds to "Slack" notification  
      - FALSE: proceeds to "Not interesting enough" node  
    - Edge cases: Missing or malformed score field, non-numeric values causing expression failures  

---

#### 2.4 Notification or No-op

- **Overview:**  
  Depending on lead qualification, either sends a Slack alert with formatted lead details or does nothing.

- **Nodes Involved:**  
  - Slack  
  - Not interesting enough

- **Node Details:**  

  - **Slack**  
    - Type: Slack node (message send)  
    - Configuration:  
      - Channel: `#interesting_leads` (selected by name)  
      - Message text: Template message including lead’s first and last name, company name, domain, location, and MadKudu top signals formatted info  
      - Credentials: Slack Bot token with permission to post messages  
    - Input: TRUE output from "if customer fit score > 60"  
    - Edge cases: Slack API token expiry, channel not found, permission errors, message formatting issues  
    - Version: Slack node v2.1  

  - **Not interesting enough**  
    - Type: No Operation (NoOp) node  
    - Role: Ends the workflow silently for leads that do not meet the score threshold  
    - Input: FALSE output from "if customer fit score > 60"  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                 | Input Node(s)            | Output Node(s)                    | Sticky Note                                                                                              |
|---------------------------|-------------------------|--------------------------------|--------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| n8n Form Trigger          | Trigger (formTrigger)    | Receives lead form submission  | None                     | Verify email with Hunter          | 👆 You can exchange this with any form you like (*e.g. Typeform, Google forms, Survey Monkey...*)       |
| Verify email with Hunter  | Hunter                  | Validates email address         | n8n Form Trigger          | Check if the email is valid       |                                                                                                        |
| Check if the email is valid | If                      | Decides on email validity       | Verify email with Hunter  | Score lead with MadKudu, Email is not valid, do nothing |                                                                                                        |
| Email is not valid, do nothing | NoOp                   | Ends workflow for invalid emails | Check if the email is valid | None                            |                                                                                                        |
| Score lead with MadKudu   | HTTP Request             | Scores lead with MadKudu API    | Check if the email is valid | if customer fit score > 60        |                                                                                                        |
| if customer fit score > 60 | If                      | Checks if lead score > 60       | Score lead with MadKudu   | Slack, Not interesting enough     | 👆 Adjust the fit as you see necessary                                                                   |
| Slack                    | Slack                   | Sends notification to Slack     | if customer fit score > 60 | None                            |                                                                                                        |
| Not interesting enough    | NoOp                    | Ends workflow for low scoring leads | if customer fit score > 60 | None                            |                                                                                                        |
| Sticky Note              | Sticky Note             | Setup instructions              | None                     | None                            | 1. Add you **MadKudu**, **Hunter**, and **Slack** credentials 2. Set the Slack channel 3. Click the Test Workflow button, enter your email and check the Slack channel 4. Activate the workflow and use the form trigger production URL to collect your leads in a smart way |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**  
   - Add a **Form Trigger** node with a unique webhook path (e.g. `0bf8840f-1cc4-46a9-86af-a3fa8da80608`).  
   - Configure the form title as "Contact us".  
   - Add a single form field labeled “What’s your business email?”.  
   - Optionally include a description "We'll get back to you soon".  
   - Save and activate the webhook.

2. **Add the Hunter Email Verifier Node:**  
   - Add a **Hunter** node with operation set to **emailVerifier**.  
   - Set the email parameter to expression referencing the form field: `={{ $json["What's your business email?"] }}`.  
   - Configure Hunter API credentials (API key).  
   - Connect the output of the Form Trigger node to this Hunter node.

3. **Add an If Node to Check Email Validity:**  
   - Add an **If** node named "Check if the email is valid".  
   - Set condition to check if the field `status` equals the string `"valid"` (case sensitive).  
   - Connect Hunter node output to this If node.

4. **Add a NoOp Node for Invalid Emails:**  
   - Add a **No Operation** node named "Email is not valid, do nothing".  
   - Connect the FALSE output of the "Check if the email is valid" If node to this NoOp node.

5. **Add HTTP Request Node to Query MadKudu:**  
   - Add an **HTTP Request** node named "Score lead with MadKudu".  
   - Set method to GET.  
   - Set URL to: `https://api.madkudu.com/v1/persons?email={{ $json.email }}`.  
   - Configure HTTP Header Authentication with your MadKudu API key.  
   - Connect the TRUE output of "Check if the email is valid" to this node.

6. **Add If Node to Check Customer Fit Score:**  
   - Add an **If** node named "if customer fit score > 60".  
   - Set condition to check if `properties.customer_fit.score` is greater than 60 (numeric).  
   - Connect the output of "Score lead with MadKudu" to this node.

7. **Add Slack Node for Notification:**  
   - Add a **Slack** node.  
   - Configure Slack API credentials (bot token with message permissions).  
   - Set the channel to the desired Slack channel by name (e.g. `#interesting_leads`).  
   - Configure text message with this template:  
     ```
     ⭐ Got a hot lead for you  {{ $json.properties.first_name }} {{ $json.properties.last_name }} from  {{ $json.company.properties.name }} ({{ $json.company.properties.domain }}) based out of {{ $json.company.properties.location.state }}, {{ $json.company.properties.location.country }}.

     {{ $('Score lead with MadKudu').item.json.properties.customer_fit.top_signals_formatted }}
     ```  
   - Connect the TRUE output of "if customer fit score > 60" to this Slack node.

8. **Add NoOp Node for Low Scoring Leads:**  
   - Add a **No Operation** node named "Not interesting enough".  
   - Connect the FALSE output of "if customer fit score > 60" to this NoOp node.

9. **Add Sticky Notes for Instructions (Optional):**  
   - Add Sticky Note nodes with setup instructions and helpful tips as per your team’s needs.

10. **Test the Workflow:**  
    - Use the test mode of the form trigger, submit a test email, and verify the Slack channel for notifications.  
    - Adjust thresholds or messages as needed.

11. **Activate the Workflow:**  
    - Once tested, activate the workflow and deploy the form URL for live lead collection.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Use your own MadKudu, Hunter.io, and Slack credentials before running the workflow.                                                                                  | Credential setup instructions in sticky notes                                                      |
| Slack messages include detailed lead info and MadKudu’s top signals to help sales prioritize outreach.                                                             | Slack node message template                                                                         |
| To adjust lead qualification sensitivity, modify the customer fit score threshold in the "if customer fit score > 60" node.                                         | Node configuration note                                                                             |
| Alternative form providers can be used by replacing the Form Trigger node.                                                                                          | Sticky note near form trigger node                                                                  |
| MadKudu API documentation: https://docs.madkudu.com/api/persons                                                                                                    | External API reference                                                                              |
| Hunter.io API documentation: https://hunter.io/api                                                                                                                 | External API reference                                                                              |
| Slack API documentation: https://api.slack.com/messaging/sending                                                                                                   | External API reference                                                                              |