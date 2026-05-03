Qualifying Appointment Requests with AI & n8n Forms

https://n8nworkflows.xyz/workflows/qualifying-appointment-requests-with-ai---n8n-forms-2580


# Qualifying Appointment Requests with AI & n8n Forms

### 1. Workflow Overview

This workflow automates the process of qualifying and scheduling appointment requests submitted via an online form, using AI to pre-qualify enquiries and streamline the booking process. It is designed for organizations aiming to reduce unnecessary appointments by assessing enquiry relevance and managing appointment approvals efficiently.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Qualification:** Collects user input through a multi-step form and uses AI text classification to determine enquiry relevance.
- **1.2 Form Progression & Terms Acceptance:** Guides the user through terms acceptance and date/time selection via segmented forms.
- **1.3 Notification & Approval Process:** Sends acknowledgment emails to the requester and initiates an approval process by notifying an admin via email, using Gmail’s approval workflow.
- **1.4 Appointment Finalization:** Depending on admin approval, creates a Google Calendar event or sends a rejection email to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Qualification

**Overview:**  
This block starts the workflow with a form trigger capturing user appointment requests and uses an AI-powered text classifier to assess if the enquiry justifies an appointment.

**Nodes Involved:**  
- n8n Form Trigger  
- OpenAI Chat Model  
- Enquiry Classifier  
- Decline (form node)  
- Terms & Conditions (form node)  
- Sticky Note3

**Node Details:**

- **n8n Form Trigger**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing user details: name, email, and enquiry text.  
  - *Configuration:* Form titled "Schedule an Appointment" with required fields. Ignores bot submissions, appends attribution, uses workflow timezone.  
  - *Input/Output:* Triggers workflow on form submission; outputs form data JSON.  
  - *Edge Cases:* Missing required fields, bot submissions (ignored).  

- **OpenAI Chat Model**  
  - *Type:* Language Model (OpenAI Chat)  
  - *Role:* Backend AI model used by the Enquiry Classifier node.  
  - *Configuration:* Uses OpenAI API credentials. No special prompt customization here (used by classifier).  
  - *Input/Output:* Accepts enquiry text; outputs classification.  
  - *Edge Cases:* API rate limits, authentication errors, network issues.

- **Enquiry Classifier**  
  - *Type:* AI Text Classifier (LangChain)  
  - *Role:* Classifies enquiry text to identify if it is "relevant enquiry" or falls to fallback category "other".  
  - *Configuration:* Categories defined with description focused on AI, automation, digital products, product engineering.  
  - *Expressions:* `inputText = {{ $json.Enquiry }}`  
  - *Input/Output:* Input from form trigger; outputs category classification.  
  - *Edge Cases:* Misclassification, AI service downtime.  
  - *Connections:*  
    - If classified relevant, proceeds to "Terms & Conditions" form.  
    - Else, proceeds to "Decline" completion form.

- **Decline (form node)**  
  - *Type:* Form (Completion)  
  - *Role:* Displays a message suggesting the user contact via direct message/email instead of scheduling.  
  - *Configuration:* Message informs enquiry may not require appointment, invites to email jim@example.com.  
  - *Edge Cases:* User leaves workflow here; no further processing.

- **Terms & Conditions**  
  - *Type:* Form  
  - *Role:* Presents terms and conditions that must be accepted before continuing.  
  - *Configuration:* Multi-select dropdown requiring user to select "I accept the terms and conditions".  
  - *Edge Cases:* User refuses to accept; handled downstream.

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Reminder to configure admin email for approval notifications.

---

#### 2.2 Form Progression & Terms Acceptance

**Overview:**  
This block manages the multi-step form experience, verifying terms acceptance and capturing requested appointment date and time.

**Nodes Involved:**  
- Has Accepted? (IF node)  
- Enter Date & Time (form node)  
- Decline1 (form node)  
- Get Form Values (Set node)  
- Sticky Note6

**Node Details:**

- **Has Accepted? (IF node)**  
  - *Type:* Conditional (IF)  
  - *Role:* Checks if user selected the acceptance of terms and conditions.  
  - *Expression:* Checks if "Please select" field starts with "I accept".  
  - *Branches:*  
    - True: continues to "Enter Date & Time" form.  
    - False: goes to Decline1 form node.

- **Enter Date & Time**  
  - *Type:* Form  
  - *Role:* Collects appointment date and time preferences.  
  - *Configuration:*  
    - Date dropdown lists next 5 weekdays (excluding weekends) dynamically using n8n expressions with Luxon DateTime.  
    - Time dropdown offers fixed time slots from 9:00 am to 6:00 pm.  
  - *Edge Cases:* User selecting no date/time (required), weekends automatically excluded.

- **Decline1 (form node)**  
  - *Type:* Form (Completion)  
  - *Role:* Similar to first decline node, informs user appointment is not scheduled.  
  - *Configuration:* Same message as Decline node.

- **Get Form Values (Set node)**  
  - *Type:* Set  
  - *Role:* Extracts and normalizes form data from previous form submissions into unified JSON with fields: name, email, enquiry, dateTime (combined date + time), and submission timestamp.  
  - *Expressions:*  
    - Combines date and time fields with Luxon `DateTime.fromFormat` to create ISO dateTime.  
  - *Edge Cases:* Date/time formatting errors if input malformed.

- **Sticky Note6**  
  - *Type:* Sticky Note  
  - *Role:* Explains rationale for splitting forms and using terms acceptance for better UX.

---

#### 2.3 Notification & Approval Process

**Overview:**  
After form completion, this block sends an acknowledgment email to the requester and triggers an admin approval subworkflow via Gmail’s "send and wait for approval" feature.

**Nodes Involved:**  
- Trigger Approval Process (Execute Workflow node)  
- Send Receipt (Gmail node)  
- Execute Workflow Trigger (Execute Workflow Trigger node)  
- Summarise Enquiry (OpenAI chain LLM node)  
- Wait for Approval (Gmail node)  
- Sticky Note7

**Node Details:**

- **Trigger Approval Process (Execute Workflow)**  
  - *Type:* Sub-workflow executor  
  - *Role:* Invokes the same workflow asynchronously for approval process without waiting for completion, passing data for admin review.  
  - *Configuration:* Mode = each item; waitForSubWorkflow = false; workflowId set to current workflow ID.  
  - *Edge Cases:* Subworkflow execution failures.

- **Send Receipt (Gmail node)**  
  - *Type:* Gmail (OAuth2)  
  - *Role:* Sends confirmation email to the requester summarizing their submission.  
  - *Configuration:*  
    - Recipient: user’s email from form data.  
    - Subject includes requested appointment date/time formatted with Luxon.  
    - HTML message includes name, email, enquiry, submission time.  
  - *Edge Cases:* Email sending failures, OAuth token expiration.

- **Execute Workflow Trigger**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for subworkflow triggered by "Trigger Approval Process". Captures context for approval.  
  - *Edge Cases:* Missing or malformed input data.

- **Summarise Enquiry (Chain LLM)**  
  - *Type:* AI Chain (LangChain)  
  - *Role:* Uses OpenAI to generate a concise summary of the enquiry for admin review.  
  - *Configuration:* Prompt instructs to summarize first 500 characters of enquiry.  
  - *Edge Cases:* AI service errors, incomplete summaries.

- **Wait for Approval (Gmail node)**  
  - *Type:* Gmail (OAuth2) with sendAndWait operation  
  - *Role:* Sends approval request email to admin with confirm/decline buttons; waits for response.  
  - *Configuration:*  
    - Recipient: admin@example.com (should be customized).  
    - Email includes appointment details, requester info, enquiry summary.  
    - Approval type: double confirmation; button label: Confirm.  
  - *Edge Cases:* Admin ignores or delays approval, email delivery failure.

- **Sticky Note7**  
  - *Type:* Sticky Note  
  - *Role:* Explains the acknowledgment and approval initiation process.

---

#### 2.4 Appointment Finalization

**Overview:**  
Based on admin response, this block creates a Google Calendar appointment or sends a rejection email to the requester.

**Nodes Involved:**  
- Has Approval? (IF node)  
- Create Appointment (Google Calendar node)  
- Send Rejection (Gmail node)  
- OpenAI Chat Model1 (OpenAI Chat)  
- Sticky Note8  
- Sticky Note

**Node Details:**

- **Has Approval? (IF node)**  
  - *Type:* Conditional  
  - *Role:* Checks if admin approved the appointment request (`$json.data.approved == true`).  
  - *Branches:*  
    - True: proceeds to create Google Calendar event.  
    - False: proceeds to send rejection email.

- **Create Appointment (Google Calendar node)**  
  - *Type:* Google Calendar (OAuth2)  
  - *Role:* Creates a calendar event for the approved appointment.  
  - *Configuration:*  
    - Start time: appointment date/time from trigger.  
    - End time: start time plus 30 minutes (fixed duration).  
    - Calendar: specific calendar ID configured (customizable).  
    - Event details: summary includes requester name and host (Jim).  
    - Attendees: requests email included.  
    - Description combines AI-generated enquiry summary and original message.  
    - Adds Google Meet conferencing.  
  - *Edge Cases:* OAuth errors, calendar API limits, invalid date/time.

- **Send Rejection (Gmail node)**  
  - *Type:* Gmail (OAuth2)  
  - *Role:* Sends rejection email to requester when appointment is declined.  
  - *Configuration:*  
    - Recipient: requester email.  
    - Subject includes appointment date/time.  
    - Message politely informs appointment cannot be scheduled.  
  - *Edge Cases:* Email failures.

- **OpenAI Chat Model1**  
  - *Type:* OpenAI Chat model node  
  - *Role:* Used to support the "Summarise Enquiry" node as backend AI model.  
  - *Configuration:* OpenAI API credentials.  
  - *Edge Cases:* Same as above AI nodes.

- **Sticky Note8**  
  - *Type:* Sticky Note  
  - *Role:* Explains the Gmail wait for approval feature and the approval/decline workflow.

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Highlights urgency to set admin email for approval emails.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                         | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                               |
|-------------------------|---------------------------------|---------------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------|
| n8n Form Trigger        | Form Trigger                    | Captures initial appointment request  | -                         | Enquiry Classifier          |                                                                                                                           |
| OpenAI Chat Model       | AI Language Model (OpenAI Chat) | Backend model for text classification | n8n Form Trigger           | Enquiry Classifier          |                                                                                                                           |
| Enquiry Classifier      | AI Text Classifier              | Classifies enquiry relevance           | n8n Form Trigger           | Terms & Conditions, Decline | [Learn more about the text classifier](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.text-classifier/) |
| Decline                 | Form (Completion)               | Decline path for irrelevant enquiries | Enquiry Classifier         | -                          |                                                                                                                           |
| Terms & Conditions      | Form                           | Terms acceptance before proceeding    | Enquiry Classifier         | Has Accepted?               |                                                                                                                           |
| Has Accepted?           | IF                            | Checks terms acceptance                | Terms & Conditions         | Enter Date & Time, Decline1 |                                                                                                                           |
| Enter Date & Time       | Form                           | Captures desired appointment schedule | Has Accepted?              | Get Form Values             | [Learn more about forms](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form)                         |
| Decline1                | Form (Completion)               | Decline path if terms not accepted    | Has Accepted?              | -                          |                                                                                                                           |
| Get Form Values         | Set                            | Normalizes and formats form data      | Enter Date & Time          | Trigger Approval Process    |                                                                                                                           |
| Trigger Approval Process| Execute Workflow               | Starts approval subworkflow            | Get Form Values            | Send Receipt                |                                                                                                                           |
| Send Receipt            | Gmail                         | Sends confirmation email to requester | Trigger Approval Process   | Form End                    |                                                                                                                           |
| Form End                | Form (Completion)               | Confirms form submission to user      | Send Receipt               | -                          |                                                                                                                           |
| Execute Workflow Trigger| Execute Workflow Trigger       | Entry point for approval subworkflow  | -                         | Summarise Enquiry           |                                                                                                                           |
| Summarise Enquiry       | AI Chain (LangChain OpenAI)    | Summarizes enquiry for admin review   | Execute Workflow Trigger   | Wait for Approval           |                                                                                                                           |
| OpenAI Chat Model1      | AI Language Model (OpenAI Chat) | Backend model for summarization       | Summarise Enquiry          | Summarise Enquiry           |                                                                                                                           |
| Wait for Approval       | Gmail (sendAndWait)            | Sends approval request to admin       | Summarise Enquiry          | Has Approval?               | [Learn more about the Waiting for Approval](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/message-operations/#send-a-message-and-wait-for-approval) |
| Has Approval?           | IF                            | Checks admin approval status           | Wait for Approval          | Create Appointment, Send Rejection |                                                                                                                           |
| Create Appointment      | Google Calendar               | Creates calendar event on approval    | Has Approval?              | -                          |                                                                                                                           |
| Send Rejection          | Gmail                         | Sends rejection email if declined     | Has Approval?              | -                          |                                                                                                                           |
| Sticky Note3            | Sticky Note                   | Highlights AI qualification block     | -                         | -                          |                                                                                                                           |
| Sticky Note6            | Sticky Note                   | Explains form splitting for UX        | -                         | -                          |                                                                                                                           |
| Sticky Note7            | Sticky Note                   | Explains acknowledgment & approval    | -                         | -                          |                                                                                                                           |
| Sticky Note8            | Sticky Note                   | Explains Gmail approval process       | -                         | -                          |                                                                                                                           |
| Sticky Note             | Sticky Note                   | Reminder to set admin email            | -                         | -                          |                                                                                                                           |
| Sticky Note9            | Sticky Note                   | Overview and usage instructions        | -                         | -                          | [Discord](https://discord.com/invite/XPKeKXeB), [Forum](https://community.n8n.io/)                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "n8n Form Trigger" node:**  
   - Type: Form Trigger  
   - Path: `schedule_appointment`  
   - Form Title: "Schedule an Appointment"  
   - Fields (all required):  
     - Text field: "Your Name" (placeholder: eg. Sam Smith)  
     - Email field: "Email" (placeholder: eg. sam@example.com)  
     - Textarea: "Enquiry" (placeholder: eg. I'm looking for...)  
   - Options: Ignore bots, append attribution, use workflow timezone.

2. **Create "OpenAI Chat Model" node:**  
   - Type: AI Language Model (OpenAI Chat)  
   - Credentials: Link your OpenAI account.  
   - Leave prompt default (used downstream).

3. **Create "Enquiry Classifier" node:**  
   - Type: AI Text Classifier (LangChain)  
   - Input Text: `={{ $json.Enquiry }}`  
   - Categories:  
     - "relevant enquiry" with description about AI, automation, digital products, product engineering.  
   - Fallback category: "other"  
   - AI Model: Connect to "OpenAI Chat Model".

4. **Connect "n8n Form Trigger" → "Enquiry Classifier".**

5. **Create "Decline" node:**  
   - Type: Form (Completion)  
   - Completion Title: "Send me a DM Instead!"  
   - Completion Message: "Thanks for your enquiry but it may not necessarily need an appointment. Please feel free to email me instead at jim@example.com."  

6. **Create "Terms & Conditions" node:**  
   - Type: Form  
   - Title: "Before we continue..."  
   - Description: Terms and Conditions text about non-binding discussions, no recording, and confirmation of understanding.  
   - Field: Dropdown, multiselect, label "Please select" with one option "I accept the terms and conditions" (required).

7. **Connect "Enquiry Classifier" node:**  
   - On category "relevant enquiry": to "Terms & Conditions".  
   - On fallback: to "Decline".

8. **Create "Has Accepted?" node:**  
   - Type: IF  
   - Condition: Check if "Please select" field starts with "I accept".  
   - True branch: continues to next form.  
   - False branch: connects to Decline1.

9. **Create "Decline1" node:**  
   - Copy configuration from "Decline".

10. **Create "Enter Date & Time" node:**  
    - Type: Form  
    - Title: "Enter a Date & Time"  
    - Description: "Please select a date and time"  
    - Define form fields via JSON:  
      - Date dropdown: next 5 weekdays excluding weekends (use Luxon expressions).  
      - Time dropdown: fixed slots from 9:00 am to 6:00 pm.  
    - Both fields required.

11. **Connect "Has Accepted?" true → "Enter Date & Time".**  
    Connect false → "Decline1".

12. **Create "Get Form Values" node:**  
    - Type: Set  
    - JSON output:  
      ```
      {
        name: $('n8n Form Trigger').first().json['Your Name'],
        email: $('n8n Form Trigger').first().json.Email,
        enquiry: $('n8n Form Trigger').first().json.Enquiry,
        dateTime: DateTime.fromFormat(`${$json.Date} ${$json.Time}`, "EEE, dd MMM t"),
        submittedAt: $('n8n Form Trigger').first().json.submittedAt,
      }
      ```
    - Connect "Enter Date & Time" → "Get Form Values".

13. **Create "Trigger Approval Process" node:**  
    - Type: Execute Workflow  
    - Mode: each item  
    - Wait for subworkflow: false  
    - Workflow ID: set to current workflow ID for subworkflow invocation.  

14. **Connect "Get Form Values" → "Trigger Approval Process".**

15. **Create "Send Receipt" node:**  
    - Type: Gmail (OAuth2)  
    - Recipient: `={{ $('Get Form Values').first().json.email }}`  
    - Subject: `=Appointment Request Received for {{ DateTime.fromISO($('Get Form Values').first().json.dateTime).format('EEE, dd MMM @ t') }}`  
    - Message (HTML): includes name, email, enquiry, submission time from "Get Form Values".  

16. **Connect "Trigger Approval Process" → "Send Receipt".**

17. **Create "Form End" node:**  
    - Type: Form (Completion)  
    - Completion Title: "Appointment Request Sent!"  
    - Completion Message: includes summary with name, date/time, enquiry.  

18. **Connect "Send Receipt" → "Form End".**

19. **In a separate workflow or subworkflow:**

    - **Create "Execute Workflow Trigger" node:**  
      - Entry point for approval subworkflow.

    - **Create "OpenAI Chat Model1" node:**  
      - OpenAI Chat for summarization AI.

    - **Create "Summarise Enquiry" node:**  
      - Type: Chain LLM (LangChain)  
      - Text: `=The enquiry is as follows:\n{{ $('Execute Workflow Trigger').first().json.enquiry.substring(0, 500) }}`  
      - Prompt: "Summarise the given enquiry"  
      - AI Model: connect to "OpenAI Chat Model1".

    - **Connect "Execute Workflow Trigger" → "Summarise Enquiry".**

    - **Create "Wait for Approval" node:**  
      - Type: Gmail (sendAndWait operation)  
      - Send to: `admin@example.com` (replace with actual admin email)  
      - Subject: "New Appointment Request!"  
      - Message: HTML including appointment date/time, name, email, enquiry summary, submission time.  
      - Approval type: double confirmation  
      - Approve label: "Confirm"

    - **Connect "Summarise Enquiry" → "Wait for Approval".**

    - **Create "Has Approval?" node:**  
      - Type: IF  
      - Condition: `$json.data.approved == true`

    - **Connect "Wait for Approval" → "Has Approval?".**

    - **Create "Create Appointment" node:**  
      - Type: Google Calendar  
      - Start: appointment date/time from trigger  
      - End: start + 30 minutes  
      - Calendar ID: specify your calendar ID  
      - Summary: "Appointment Scheduled - {name} & Jim"  
      - Attendees: requester email  
      - Description: combines AI summary and original enquiry  
      - Conference: Google Meet enabled  
      - Credentials: Google Calendar OAuth2.

    - **Create "Send Rejection" node:**  
      - Type: Gmail  
      - Recipient: requester email  
      - Subject: "Appointment Request Rejected for {date/time}"  
      - Message: polite rejection message.

    - **Connect "Has Approval?" true → "Create Appointment".**  
      Connect false → "Send Rejection".

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Learn more about the text classifier node to understand AI-based enquiry classification.                                           | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.text-classifier/       |
| Learn about the n8n form node to build multi-step forms for better UX.                                                             | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form                                      |
| Gmail node "send and wait for approval" enables human-in-the-loop workflows with approval buttons.                                 | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/message-operations/#send-a-message-and-wait-for-approval |
| Join n8n community for help: Discord and Forum.                                                                                     | Discord: https://discord.com/invite/XPKeKXeB; Forum: https://community.n8n.io/                               |
| Reminder: Replace placeholder admin email (`admin@example.com`) with your actual administrative email to receive approvals.       | Workflow Sticky Note                                                                                         |
| Google Calendar and Gmail credentials must be configured with OAuth2 for calendar event creation and email sending capabilities.    | Credential setup in n8n                                                                                        |

---

This reference document fully describes the workflow's structure, nodes, configurations, and logic to allow both human developers and AI agents to understand, reproduce, and extend the automation for qualifying appointment requests via AI-powered forms and approval processes.