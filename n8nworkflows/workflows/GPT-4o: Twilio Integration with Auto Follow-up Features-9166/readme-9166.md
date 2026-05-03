GPT-4o: Twilio Integration with Auto Follow-up Features

https://n8nworkflows.xyz/workflows/gpt-4o--twilio-integration-with-auto-follow-up-features-9166


# GPT-4o: Twilio Integration with Auto Follow-up Features

### 1. Workflow Overview

This workflow, titled **"GPT-4o: Twilio Integration with Auto Follow-up Features"**, automates the handling of missed calls received via Twilio. It is designed primarily for businesses seeking to recover lost revenue from missed calls by automatically analyzing call context, checking CRM records, generating personalized AI-driven responses, sending SMS and/or email follow-ups, logging interactions, notifying sales teams via Slack, and scheduling automatic follow-ups if no booking activity is detected within 24 hours.

The workflow’s architecture is logically grouped into these blocks:

- **1.1 Missed Call Detection and Context Analysis:** Listens for missed calls via Twilio webhook and filters out answered calls while extracting context and urgency.
- **1.2 CRM Lookup and Data Merging:** Queries CRM for existing contact info and merges call data with customer history and prioritization logic.
- **1.3 AI Response Generation:** Uses OpenAI GPT-4o via LangChain integration to create personalized SMS and email responses based on customer and call context.
- **1.4 Message Preparation and Sending:** Prepares messages with booking links and conditionally sends SMS and/or email based on AI recommendation and contact data.
- **1.5 Results Aggregation and Tracking:** Merges SMS and email send results, tracks which channels were used, and timestamps message dispatch.
- **1.6 Logging and Notification:** Logs all activity to Airtable CRM and notifies the sales team on Slack with detailed call and customer data.
- **1.7 Follow-up Automation:** Waits 24 hours, checks booking status via CRM API, and sends an automated follow-up SMS if no booking was made.

---

### 2. Block-by-Block Analysis

#### 2.1 Missed Call Detection and Context Analysis

- **Overview:**  
  This block triggers on a missed call event from Twilio, extracts detailed call information, filters out calls that were answered, and assesses the urgency based on business hours and timing.

- **Nodes Involved:**  
  - Missed Call Detected2 (Twilio Trigger)  
  - Analyze Call Context2 (Code)

- **Node Details:**  

  - **Missed Call Detected2**  
    - Type: Twilio Trigger  
    - Role: Listens to Twilio voice insights webhook for completed call summaries.  
    - Parameters: Watches for event `com.twilio.voice.insights.call-summary.complete`.  
    - Credentials: Uses Twilio API credentials.  
    - Connections: Output connected to Analyze Call Context2.  
    - Potential Failures: Network errors, Twilio auth failures, webhook misconfiguration.

  - **Analyze Call Context2**  
    - Type: Code node  
    - Role: Parses call details, determines if call was answered, extracts caller/receiver numbers, call timestamp, calculates urgency score.  
    - Key Logic:  
      - Skips calls with duration > 0 (answered).  
      - Derives business hours (9am–5pm) and weekend status.  
      - Assigns urgency score higher for after-hours calls.  
    - Inputs: JSON from Twilio trigger.  
    - Outputs: Structured JSON with enriched call data and urgency.  
    - Edge Cases: Missing or malformed timestamps, unexpected call status values.

#### 2.2 CRM Lookup and Data Merging

- **Overview:**  
  Queries CRM system for existing contact by phone number, then merges CRM data with call context, applying prioritization and customer segmentation.

- **Nodes Involved:**  
  - Check Existing Contact2 (HTTP Request)  
  - Merge Customer Data2 (Code)

- **Node Details:**  

  - **Check Existing Contact2**  
    - Type: HTTP Request  
    - Role: Searches CRM API endpoint for contact data using caller phone number.  
    - Parameters: URL template uses `caller_number` from previous node.  
    - Authentication: HTTP Header Auth credential (configurable).  
    - Outputs: CRM JSON data or empty if no contact found.  
    - Failure Modes: HTTP errors, auth failures, API downtime.

  - **Merge Customer Data2**  
    - Type: Code node  
    - Role: Combines call context and CRM data.  
    - Logic:  
      - Detects existing customer, extracts name, email, status, purchase history, notes.  
      - Sets priority level: High if lifetime value > 1000, else medium or new.  
      - Assigns follow-up urgency accordingly.  
    - Input: Outputs of Analyze Call Context2 and Check Existing Contact2.  
    - Output: Enriched JSON with merged customer and call data.  
    - Edge Cases: Missing CRM fields, null values.

#### 2.3 AI Response Generation

- **Overview:**  
  Generates personalized AI responses (SMS and email) using GPT-4o via LangChain. The prompt includes customer and call context to tailor tone, urgency, and content.

- **Nodes Involved:**  
  - Generate AI Response2 (LangChain Chain LLM)  
  - OpenAI Chat Model8 (LangChain OpenAI LLM)  
  - Structured Output Parser8 (LangChain Output Parser)

- **Node Details:**  

  - **Generate AI Response2**  
    - Type: LangChain Chain LLM node  
    - Role: Defines prompt with customer info, call time, priority, and requests two message types: SMS and email.  
    - Output: JSON structured with SMS message, email subject/body, tone, recommended channel.  
    - Input: Merged customer data.  
    - Output connections: Linked to OpenAI Chat Model8 (LLM execution) and Structured Output Parser8 (output parsing).  
    - Edge Cases: Invalid or incomplete input data causing prompt errors or unexpected output.

  - **OpenAI Chat Model8**  
    - Type: LangChain OpenAI LLM node  
    - Role: Executes the AI prompt using GPT-4o with temperature 0.7 and max tokens 500.  
    - Input: Prompt from Generate AI Response2.  
    - Output: Raw AI completion.  
    - Potential Issues: API rate limits, auth failures, model errors.

  - **Structured Output Parser8**  
    - Type: LangChain Output Parser  
    - Role: Parses AI output into defined JSON schema with validation and auto-fix enabled.  
    - Schema: Requires sms, email_subject, email_body, tone, recommended_channel fields.  
    - Output: Parsed structured JSON.  
    - Failure Modes: Parsing errors if AI output deviates from schema.

#### 2.4 Message Preparation and Sending

- **Overview:**  
  Prepares messages by inserting booking links, then conditionally sends SMS and/or email based on AI recommendation and contact info presence.

- **Nodes Involved:**  
  - Prepare Messages2 (Code)  
  - Send SMS?2 (If)  
  - Send SMS2 (Twilio)  
  - Send Email?2 (If)  
  - Send Email2 (Email Send)

- **Node Details:**  

  - **Prepare Messages2**  
    - Type: Code node  
    - Role: Inserts booking link into AI-generated SMS and email messages using customer name and phone as URL query params.  
    - Booking URL placeholder: `https://cal.com/your-business` (replaceable).  
    - Outputs enriched JSON with final messages and metadata.  
    - Edge Cases: Missing customer name/phone causing malformed URLs.

  - **Send SMS?2**  
    - Type: If node  
    - Role: Checks if AI recommends SMS or both channels for sending SMS.  
    - Conditions: recommended_channel == 'sms' OR 'both'.  
    - Outputs: True branch to Send SMS2.

  - **Send SMS2**  
    - Type: Twilio node  
    - Role: Sends SMS message to caller via Twilio using configured 'from' number.  
    - Parameters: To = caller_number, From = Twilio number (replace `+15551234567`), Message = sms_message.  
    - Credentials: Twilio API.  
    - Outputs: Twilio response (messageSid).  
    - Failures: Twilio sending errors, invalid phone numbers.

  - **Send Email?2**  
    - Type: If node  
    - Role: Checks if customer email exists AND AI recommends email channel (or both).  
    - Conditions: customer_email not empty AND recommended_channel contains 'email'.  
    - Outputs: True branch to Send Email2.

  - **Send Email2**  
    - Type: Email Send node  
    - Role: Sends personalized email with subject and body to customer.  
    - Parameters: ToEmail = customer_email, FromEmail = business email (replace `your-business@example.com`), Subject/Body from AI response.  
    - Credentials: SMTP account.  
    - Failures: SMTP connection issues, invalid email addresses.

#### 2.5 Results Aggregation and Tracking

- **Overview:**  
  Combines SMS and email send results, tracks which channels were used, timestamps sending, and prepares data for logging.

- **Nodes Involved:**  
  - Merge Results (Merge)  
  - Track Results (Code)

- **Node Details:**  

  - **Merge Results**  
    - Type: Merge node (mode: default)  
    - Role: Combines outputs of SMS send and Email send (both may run in parallel).  
    - Inputs: From Send SMS2 and Send Email2.  
    - Outputs: Combined array of results.

  - **Track Results**  
    - Type: Code node  
    - Role:  
      - Detects if SMS was sent by presence of Twilio sid/messageSid.  
      - Detects if email was sent by presence of accepted/messageId.  
      - Adds flags sms_sent, email_sent, and timestamp messages_sent_at.  
      - Merges with customer data from Prepare Messages2 node.  
    - Output: Aggregated JSON for logging and notifications.

#### 2.6 Logging and Notification

- **Overview:**  
  Logs call and messaging data into Airtable CRM, then sends a detailed notification to Slack sales channel.

- **Nodes Involved:**  
  - Log to CRM2 (Airtable)  
  - Notify Sales Team3 (Slack)

- **Node Details:**  

  - **Log to CRM2**  
    - Type: Airtable node  
    - Role: Upserts record in Airtable base/table with call and messaging details.  
    - Columns mapped: Status ("Pending Response"), Priority, SMS_Sent, Email_Sent, Booking_Link, Caller_Number, Customer_Name, Urgency_Score, Follow_Up_Date (set 24h later).  
    - Credentials: Airtable API Token.  
    - Failure Modes: Airtable API limits, invalid base/table IDs.

  - **Notify Sales Team3**  
    - Type: Slack node  
    - Role: Sends formatted message to Slack channel about missed call, customer data, and auto-response status.  
    - Channel ID: Configured (replace placeholder).  
    - Message includes call time, customer status, priority, previous purchases, booking link, and action required.  
    - Credentials: Slack OAuth2.  
    - Failures: Slack API errors, invalid channel ID.

#### 2.7 Follow-up Automation

- **Overview:**  
  Waits 24 hours, checks CRM for booking status, and if no booking was made, sends a follow-up SMS to encourage customer engagement.

- **Nodes Involved:**  
  - Wait 24 Hours2 (Wait)  
  - Check if Booked2 (HTTP Request)  
  - No Response?2 (If)  
  - Send Follow-up SMS2 (Twilio)

- **Node Details:**  

  - **Wait 24 Hours2**  
    - Type: Wait node  
    - Role: Pauses workflow execution for 24 hours before checking booking status.  
    - Parameters: 24 hours.  
    - Failures: Workflow timeout if exceeded.

  - **Check if Booked2**  
    - Type: HTTP Request  
    - Role: Queries CRM API to retrieve booking count/status for caller_number.  
    - URL templated with caller_number.  
    - Authentication: HTTP Header Auth credential.  
    - Output: JSON with booking_count field.  
    - Failures: API downtime, auth errors.

  - **No Response?2**  
    - Type: If node  
    - Role: Checks if booking_count is zero (no booking).  
    - Condition: booking_count == 0  
    - True branch: Send Follow-up SMS2.

  - **Send Follow-up SMS2**  
    - Type: Twilio node  
    - Role: Sends a friendly follow-up SMS to customer encouraging booking.  
    - Parameters: To = caller_number, From = Twilio number (replace `+15551234567`), Message includes customer name, booking link, and business contact.  
    - Credentials: Twilio API.  
    - Failures: Twilio send errors, invalid phone.

---

### 3. Summary Table

| Node Name              | Node Type                   | Functional Role                          | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                |
|------------------------|-----------------------------|----------------------------------------|-----------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Missed Call Detected2   | Twilio Trigger              | Trigger on missed calls                 | —                           | Analyze Call Context2           | See Sticky Note9 for full setup instructions and overview.                                                                |
| Analyze Call Context2   | Code                        | Extract and analyze call details        | Missed Call Detected2        | Check Existing Contact2         |                                                                                                                            |
| Check Existing Contact2 | HTTP Request                | Query CRM for contact by phone          | Analyze Call Context2        | Merge Customer Data2            |                                                                                                                            |
| Merge Customer Data2    | Code                        | Merge CRM data with call context        | Check Existing Contact2      | Generate AI Response2           |                                                                                                                            |
| Generate AI Response2   | LangChain Chain LLM         | Generate personalized AI SMS/email      | Merge Customer Data2         | OpenAI Chat Model8, Structured Output Parser8 |                                                                                                                            |
| OpenAI Chat Model8      | LangChain OpenAI LLM        | Execute GPT-4o prompt                    | Generate AI Response2        | Structured Output Parser8       |                                                                                                                            |
| Structured Output Parser8| LangChain Output Parser    | Parse AI response into JSON schema      | OpenAI Chat Model8           | Generate AI Response2           |                                                                                                                            |
| Prepare Messages2       | Code                        | Insert booking links into messages      | Generate AI Response2        | Send SMS?2, Send Email?2        |                                                                                                                            |
| Send SMS?2             | If                          | Condition to send SMS                    | Prepare Messages2            | Send SMS2                      |                                                                                                                            |
| Send SMS2              | Twilio                      | Send SMS via Twilio                      | Send SMS?2                  | Merge Results                  | Replace `+15551234567` with your Twilio phone number (2 SMS nodes).                                                        |
| Send Email?2           | If                          | Condition to send Email                  | Prepare Messages2            | Send Email2                   |                                                                                                                            |
| Send Email2            | Email Send                  | Send email via SMTP                      | Send Email?2                | Merge Results                  | Replace `your-business@example.com` with your business email.                                                              |
| Merge Results          | Merge                       | Merge SMS and Email send results        | Send SMS2, Send Email2       | Track Results                  |                                                                                                                            |
| Track Results          | Code                        | Track which channels messages were sent | Merge Results                | Log to CRM2                   |                                                                                                                            |
| Log to CRM2            | Airtable                    | Log call and messaging data to Airtable | Track Results               | Notify Sales Team3             | Replace Airtable Base & Table IDs.                                                                                         |
| Notify Sales Team3     | Slack                       | Notify sales team via Slack              | Log to CRM2                 | Wait 24 Hours2                | Replace Slack Channel ID.                                                                                                   |
| Wait 24 Hours2         | Wait                        | Pause for 24 hours before follow-up     | Notify Sales Team3           | Check if Booked2              |                                                                                                                            |
| Check if Booked2       | HTTP Request                | Check CRM if booking was made            | Wait 24 Hours2             | No Response?2                 | Replace CRM API endpoint and set HTTP Header Auth credential.                                                              |
| No Response?2          | If                          | Check if no booking was made             | Check if Booked2            | Send Follow-up SMS2           |                                                                                                                            |
| Send Follow-up SMS2    | Twilio                      | Send follow-up SMS if no booking         | No Response?2               | —                            | Replace `+15551234567` with your Twilio phone number; customize message text and business contact info.                     |
| Sticky Note9           | Sticky Note                 | Setup instructions and workflow overview | —                          | —                            | Contains detailed multi-step configuration and usage instructions for Twilio, Slack, Airtable, CRM, and credentials setup. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Twilio Trigger Node**  
   - Node Type: Twilio Trigger  
   - Name: "Missed Call Detected2"  
   - Configure webhook to listen for event: `com.twilio.voice.insights.call-summary.complete`  
   - Set Twilio credentials with Account SID and Auth Token.

2. **Add Code Node to Analyze Call Context**  
   - Node Type: Code  
   - Name: "Analyze Call Context2"  
   - Input: Output from Twilio Trigger  
   - Paste JavaScript logic to parse call data, skip answered calls, extract caller info, determine business hours, urgency score.

3. **Add HTTP Request Node for CRM Lookup**  
   - Node Type: HTTP Request  
   - Name: "Check Existing Contact2"  
   - URL: `https://your-crm-api.com/contacts/search?phone={{ $json.caller_number }}`  
   - Authentication: HTTP Header Auth (configure with appropriate CRM API key)  
   - Connect input from Analyze Call Context2.

4. **Add Code Node to Merge Customer Data**  
   - Node Type: Code  
   - Name: "Merge Customer Data2"  
   - Input: Outputs from Analyze Call Context2 and Check Existing Contact2  
   - Logic: Merge call and CRM data, assign priority and urgency.

5. **Add LangChain Chain LLM Node for AI Response Generation**  
   - Node Type: LangChain Chain LLM  
   - Name: "Generate AI Response2"  
   - Input: Merged customer data  
   - Prompt: Use personalized prompt for SMS (160 chars max) and email (200 words) with placeholders and tone instructions.  
   - Connect output to OpenAI Chat Model8.

6. **Add LangChain OpenAI LLM Node**  
   - Node Type: LangChain OpenAI LLM  
   - Name: "OpenAI Chat Model8"  
   - Model: GPT-4o  
   - Parameters: Max tokens 500, Temperature 0.7  
   - Input: Prompt from Generate AI Response2.

7. **Add LangChain Output Parser Node**  
   - Node Type: Structured Output Parser  
   - Name: "Structured Output Parser8"  
   - Schema: JSON schema requiring sms, email_subject, email_body, tone, recommended_channel  
   - Connect input from OpenAI Chat Model8  
   - Connect output back to Generate AI Response2 for parsed output.

8. **Add Code Node to Prepare Messages**  
   - Node Type: Code  
   - Name: "Prepare Messages2"  
   - Input: AI parsed response and merged customer data  
   - Logic: Insert booking link URL with encoded customer name and phone into SMS and email body.

9. **Add IF Node to Check SMS Sending Condition**  
   - Node Type: If  
   - Name: "Send SMS?2"  
   - Condition: recommended_channel equals 'sms' OR 'both'  
   - True branch: Connect to Send SMS2 node.

10. **Add Twilio Node to Send SMS**  
    - Node Type: Twilio  
    - Name: "Send SMS2"  
    - To: `{{ $json.caller_number }}`  
    - From: your Twilio phone number (replace `+15551234567`)  
    - Message: `{{ $json.sms_message }}`  
    - Credentials: Twilio API

11. **Add IF Node to Check Email Sending Condition**  
    - Node Type: If  
    - Name: "Send Email?2"  
    - Conditions: customer_email not empty AND recommended_channel contains 'email'  
    - True branch: Connect to Send Email2 node.

12. **Add Email Send Node**  
    - Node Type: Email Send  
    - Name: "Send Email2"  
    - ToEmail: `{{ $json.customer_email }}`  
    - FromEmail: your business email (e.g., `your-business@example.com`)  
    - Subject: `{{ $json.email_subject }}`  
    - Body: `{{ $json.email_body }}`  
    - Credentials: SMTP configured for your sending email account

13. **Add Merge Node to Combine SMS and Email Results**  
    - Node Type: Merge  
    - Name: "Merge Results"  
    - Input 1: From Send SMS2  
    - Input 2: From Send Email2

14. **Add Code Node to Track Sent Results**  
    - Node Type: Code  
    - Name: "Track Results"  
    - Input: Merge Results output and Prepare Messages2 JSON  
    - Logic: Detect SMS/email sent status and timestamp.

15. **Add Airtable Node to Log to CRM**  
    - Node Type: Airtable  
    - Name: "Log to CRM2"  
    - Base ID: Replace with your Airtable Base ID  
    - Table ID: Replace with Airtable Table ID  
    - Map fields: Status, Priority, SMS_Sent, Email_Sent, Booking_Link, Caller_Number, Customer_Name, Urgency_Score, Follow_Up_Date (set +24h)  
    - Credentials: Airtable API token  
    - Input: Track Results output

16. **Add Slack Node to Notify Sales Team**  
    - Node Type: Slack  
    - Name: "Notify Sales Team3"  
    - Channel ID: Replace with Slack sales channel ID  
    - Message: Template including customer name, call time, priority, booking link, SMS/email status  
    - Credentials: Slack OAuth2  
    - Input: Log to CRM2 output

17. **Add Wait Node for 24 Hours Delay**  
    - Node Type: Wait  
    - Name: "Wait 24 Hours2"  
    - Duration: 24 hours  
    - Input: Notify Sales Team3 output

18. **Add HTTP Request to Check Booking Status**  
    - Node Type: HTTP Request  
    - Name: "Check if Booked2"  
    - URL: `https://your-crm-api.com/contacts/{{ $json.caller_number }}/bookings`  
    - Authentication: HTTP Header Auth (CRM API key)  
    - Input: Wait 24 Hours2 output

19. **Add IF Node to Check for No Booking**  
    - Node Type: If  
    - Name: "No Response?2"  
    - Condition: booking_count == 0  
    - True branch: Connect to Send Follow-up SMS2 node

20. **Add Twilio Node to Send Follow-up SMS**  
    - Node Type: Twilio  
    - Name: "Send Follow-up SMS2"  
    - To: `{{ $json.caller_number }}`  
    - From: your Twilio number (replace `+15551234567`)  
    - Message: Friendly follow-up text with booking link and business contact info  
    - Credentials: Twilio API

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------|
| **Setup instructions, integration details, and credentials needed are comprehensively documented in the Sticky Note9 node within the workflow for easy reference.**                                                                                  | Sticky Note9 within workflow       |
| Replace placeholders with your actual Twilio phone number, SMTP email, Slack channel ID, Airtable base/table IDs, and CRM API URLs before activating the workflow.                                                                                  | Configuration requirement          |
| Booking links are currently templated for Cal.com but can be replaced with other scheduling tools like Calendly by updating the booking URL in Prepare Messages2 and follow-up SMS text.                                                            | Booking URL customization          |
| This workflow demonstrates a practical use of GPT-4o via LangChain integration for personalized customer interaction automation in missed call revenue recovery scenarios.                                                                             | AI integration notes               |
| Slack notification messages include rich customer context and actionable instructions to sales teams to prioritize callbacks.                                                                                                                      | Slack usage                       |
| Ensure all credentials (Twilio, OpenAI, Slack, Airtable, SMTP, CRM API) are properly set up in n8n Credentials manager before activating the workflow.                                                                                                | Credential setup reminder          |
| The workflow auto-configures the Twilio webhook upon activation; ensure your Twilio account allows incoming call webhook setup.                                                                                                                     | Twilio webhook configuration       |

---

*Disclaimer:*  
The text provided is sourced exclusively from an automated workflow created with n8n, a workflow automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.