Generate Smart Calendly Notifications for Outlook and Slack using GPT-4

https://n8nworkflows.xyz/workflows/generate-smart-calendly-notifications-for-outlook-and-slack-using-gpt-4-7305


# Generate Smart Calendly Notifications for Outlook and Slack using GPT-4

### 1. Workflow Overview

This workflow automates the generation and delivery of smart, AI-enhanced notifications for new Calendly bookings. When someone schedules an appointment via Calendly, the workflow extracts relevant details, uses GPT-4 to generate a customized Outlook HTML email and Slack message, then sends those notifications to the appropriate recipients.

The workflow is logically divided into these blocks:

- **1.1 Event Trigger and Data Extraction:** Listens for new Calendly invitee events and extracts key booking details.
- **1.2 AI Message Generation:** Uses the OpenAI GPT-4 model to create personalized notification messages formatted for Outlook email and Slack.
- **1.3 Notification Delivery:** Sends the generated email via Microsoft Outlook and posts the message to a Slack channel.
- **1.4 Utility and Documentation:** Sticky notes provide setup instructions and contact info for support.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Trigger and Data Extraction

- **Overview:**  
  This block waits for new invitee creation events from Calendly, then extracts and formats crucial booking information like invitee name, email, responses to questions, and event start time for downstream processing.

- **Nodes Involved:**  
  - `Calendly Event`  
  - `Edit Fields`

- **Node Details:**

  1. **Calendly Event**  
     - *Type & Role:* Calendly Trigger node; listens to Calendly webhook events.  
     - *Configuration:* Set to trigger on `invitee.created` event only. Credentials use a configured Calendly API token.  
     - *Expressions:* None, receives JSON payload from Calendly API.  
     - *Connections:* Outputs to `Edit Fields`.  
     - *Edge Cases:*  
       - Webhook failures or misconfiguration can cause no triggers.  
       - API rate limits or revoked tokens can cause auth errors.  
       - Unexpected payload changes from Calendly API could break parsing downstream.

  2. **Edit Fields**  
     - *Type & Role:* Set node; extracts and reassigns relevant fields from the raw Calendly payload for easier reference.  
     - *Configuration:* Assigns four fields:  
       - `payload.email` = invitee’s email  
       - `payload.questions_and_answers[0].answer` = first question’s answer  
       - `payload.name` = invitee’s name  
       - `payload.scheduled_event.start_time` = event start time  
     - *Expressions:* Uses expressions like `={{ $json.payload.email }}` to pull data from input JSON.  
     - *Connections:* Outputs to `Email Generator`.  
     - *Edge Cases:*  
       - Missing or malformed fields in Calendly payload can cause empty or invalid values.  
       - Assumes at least one question answer exists; otherwise may fail or produce nulls.

---

#### 2.2 AI Message Generation

- **Overview:**  
  This block leverages OpenAI’s GPT-4 model via Langchain nodes to generate customized notification messages formatted as HTML email content and Slack message text. It uses a structured output parser to enforce JSON output format.

- **Nodes Involved:**  
  - `OpenAI Chat Model`  
  - `Structured Output Parser`  
  - `Email Generator`

- **Node Details:**

  1. **OpenAI Chat Model**  
     - *Type & Role:* Langchain OpenAI chat model node; interfaces with GPT-4.  
     - *Configuration:* Uses model `gpt-4.1-mini` (a GPT-4 variant). No additional options set. Credentials use OpenAI API key.  
     - *Expressions:* None directly; it receives prompt input from `Email Generator`.  
     - *Connections:* Outputs to `Email Generator` via AI language model input.  
     - *Edge Cases:*  
       - API rate limits, quota exhaustion, or invalid API key cause failures.  
       - Model response latency or downtime can delay workflow.

  2. **Structured Output Parser**  
     - *Type & Role:* Langchain output parser node; ensures AI output matches expected JSON schema.  
     - *Configuration:* Example JSON schema requires two fields: `email` (HTML string) and `slack` (plain text string).  
     - *Connections:* Outputs to `Email Generator` as AI output parser input.  
     - *Edge Cases:*  
       - Malformed or unexpected AI responses may cause parsing failures.  
       - Schema mismatch leads to workflow errors or empty outputs.

  3. **Email Generator**  
     - *Type & Role:* Langchain agent node; coordinates prompt creation and AI output parsing.  
     - *Configuration:*  
       - Prompt text includes invitee name, question answer, and start time.  
       - System message instructs the AI to produce an email in HTML and Slack text message in JSON format.  
       - Output parser enabled to enforce structured response.  
     - *Expressions:* Constructs prompt dynamically with fields from `Edit Fields`.  
     - *Connections:* Sends final outputs to both `Send a message` (email) and `Slack Message` nodes.  
     - *Edge Cases:*  
       - Incorrect prompt formatting or missing data fields can degrade AI output quality.  
       - Parsing errors propagate here from output parser.

---

#### 2.3 Notification Delivery

- **Overview:**  
  Sends the AI-generated notifications: an HTML email via Outlook to the invitee’s email address, and a Slack message to a specified channel.

- **Nodes Involved:**  
  - `Send a message` (Microsoft Outlook)  
  - `Slack Message`

- **Node Details:**

  1. **Send a message**  
     - *Type & Role:* Microsoft Outlook node; sends email messages.  
     - *Configuration:*  
       - Subject fixed as “Calendly Details”  
       - Body content uses AI-generated HTML email (`output.email`).  
       - Recipient email dynamically set from invitee email extracted earlier.  
       - Body content type set to HTML for rich formatting.  
       - Uses OAuth2 credentials for Outlook account.  
     - *Connections:* Terminal node (no outputs).  
     - *Edge Cases:*  
       - OAuth token expiry or revocation can cause auth failures.  
       - Invalid email address or mail server issues lead to delivery failure.  
       - Large email content may cause timeout.

  2. **Slack Message**  
     - *Type & Role:* Slack node; posts message to Slack channel.  
     - *Configuration:*  
       - Message text dynamically set from AI output (`output.slack`).  
       - Channel fixed to `#leads`.  
       - Uses OAuth2 credentials for Slack workspace access.  
     - *Connections:* Terminal node (no outputs).  
     - *Edge Cases:*  
       - Slack API rate limits or token issues cause failures.  
       - Incorrect channel name or permissions prevent posting.

---

#### 2.4 Utility and Documentation

- **Overview:**  
  Provides user-facing documentation and support contact information within the workflow for easier setup and customization.

- **Nodes Involved:**  
  - `Sticky Note16`  
  - `Sticky Note`

- **Node Details:**

  1. **Sticky Note16**  
     - *Type & Role:* Sticky note node; displays contact info for support.  
     - *Content:* Email and LinkedIn contact of workflow author.  
     - *Edge Cases:* None (informational only).

  2. **Sticky Note**  
     - *Type & Role:* Sticky note node; provides detailed step-by-step setup instructions for credentials and nodes.  
     - *Content:*  
       - Calendly API setup instructions.  
       - Microsoft Outlook OAuth2 credential guidance.  
       - Slack OAuth2 setup notes.  
       - OpenAI API key configuration.  
       - Summary of key nodes and their roles.  
     - *Edge Cases:* None (informational only).

---

### 3. Summary Table

| Node Name         | Node Type                             | Functional Role                         | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                        |
|-------------------|-------------------------------------|---------------------------------------|----------------------|-----------------------|--------------------------------------------------------------------------------------------------------------------|
| Calendly Event    | Calendly Trigger                    | Trigger on new Calendly invitee event | -                    | Edit Fields           | See step-by-step setup instructions for Calendly API configuration                                                 |
| Edit Fields       | Set                                | Extracts key booking details           | Calendly Event       | Email Generator        | See step-by-step setup instructions                                                                                 |
| Email Generator   | Langchain Agent                    | Generates AI email and Slack messages  | Edit Fields, OpenAI Chat Model, Structured Output Parser | Send a message, Slack Message | See step-by-step setup instructions; uses GPT-4 to format messages                                                  |
| OpenAI Chat Model | Langchain OpenAI Chat Model        | Provides GPT-4 AI model for generation | -                    | Email Generator (AI languageModel input) | See step-by-step setup instructions for OpenAI API key                                                             |
| Structured Output Parser | Langchain Output Parser        | Parses AI output to JSON format         | -                    | Email Generator (AI outputParser input) | See step-by-step setup instructions                                                                                 |
| Send a message    | Microsoft Outlook                  | Sends formatted HTML email              | Email Generator       | -                     | See step-by-step setup instructions for Outlook OAuth2 credentials                                                 |
| Slack Message     | Slack                             | Posts message to Slack channel          | Email Generator       | -                     | See step-by-step setup instructions for Slack OAuth2 credentials                                                  |
| Sticky Note16     | Sticky Note                       | Contact info for support                 | -                    | -                     | 📧 robert@ynteractive.com; LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/                             |
| Sticky Note       | Sticky Note                       | Setup instructions and workflow summary | -                    | -                     | Detailed setup steps for Calendly, Outlook, Slack, OpenAI, and node roles                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Calendly Trigger Node**  
   - Add node: *Calendly Trigger*  
   - Set event to `invitee.created`  
   - Configure credentials with your Calendly API personal access token  
   - Position node as the workflow start point

2. **Add Set Node to Extract Fields**  
   - Add node: *Set*  
   - Create assignments extracting:  
     - `payload.email` from `{{$json.payload.email}}`  
     - `payload.questions_and_answers[0].answer` from `{{$json.payload.questions_and_answers[0].answer}}`  
     - `payload.name` from `{{$json.payload.name}}`  
     - `payload.scheduled_event.start_time` from `{{$json.payload.scheduled_event.start_time}}`  
   - Connect output of Calendly Trigger to this Set node

3. **Configure OpenAI Chat Model Node**  
   - Add node: *Langchain OpenAI Chat Model*  
   - Select model `gpt-4.1-mini` (or GPT-4 if available)  
   - Provide OpenAI API credentials (API key)  
   - No additional options needed  
   - Connect this node's output to the AI language model input of the Email Generator node (next step)

4. **Add Structured Output Parser Node**  
   - Add node: *Langchain Output Parser Structured*  
   - Define JSON schema example:  
     ```json
     {
       "email": "html email",
       "slack": "Slack Message"
     }
     ```  
   - Connect this node's output to the AI output parser input of Email Generator node

5. **Create Email Generator Node**  
   - Add node: *Langchain Agent*  
   - Set prompt text to:  
     ```
     Name:  {{ $json.payload.name }} About:  {{ $json.payload.questions_and_answers[0].answer }} Start Time: {{ $json.payload.scheduled_event.start_time }}
     ```  
   - Add system message:  
     ```
     You are a helpful assistant. Write an email and a slack message notifying me that someone booked a calendly appt with me. Output the slack message in text. and the outlook message in html.

     output like this.

     {
       "email": "html email",
       "slack": "Slack Message"
     }
     ```  
   - Enable output parser  
   - Connect input from Set node and the two Langchain nodes (OpenAI Chat Model and Structured Output Parser)

6. **Add Microsoft Outlook Node to Send Email**  
   - Add node: *Microsoft Outlook*  
   - Set subject: `Calendly Details`  
   - Set body content to `{{$json.output.email}}`  
   - Set recipient to `={{$('Edit Fields').item.json.payload.email}}`  
   - Set body content type to HTML  
   - Configure OAuth2 credentials for Microsoft Outlook  
   - Connect this node’s input from Email Generator node

7. **Add Slack Node to Post Message**  
   - Add node: *Slack*  
   - Set message text to `={{ $json.output.slack }}`  
   - Set channel to `#leads` (or your preferred channel)  
   - Configure OAuth2 credentials for Slack  
   - Connect this node’s input from Email Generator node

8. **Add Sticky Notes (Optional but Recommended)**  
   - Create sticky notes for:  
     - Contact details (email and LinkedIn)  
     - Detailed setup instructions covering Calendly, Outlook, Slack, OpenAI, and node roles

9. **Connect Workflow**  
   - Connect nodes as follows:  
     - Calendly Event → Edit Fields → Email Generator → [Send a message (Outlook), Slack Message]  
     - OpenAI Chat Model and Structured Output Parser nodes connect to Email Generator’s AI inputs

10. **Test Workflow**  
    - Simulate a Calendly booking or use pinned example data  
    - Verify email and Slack notifications are correctly formatted and delivered

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Contact for help or customization: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/             | Support contact provided in Sticky Note16                                                               |
| Detailed step-by-step instructions for setting up Calendly, Microsoft Outlook OAuth2, Slack OAuth2, and OpenAI credentials included | Provided in Sticky Note with workflow setup instructions                                                 |
| Workflow uses GPT-4 AI to create custom notification content, improving message personalization and formatting                      | AI agent node configured to produce JSON with HTML email and Slack message                               |
| Calendly event listened for is `invitee.created` to ensure notifications on new bookings only                                        | Trigger node configuration                                                                                |
| Slack messages posted to the `#leads` channel, adjust as needed for your workspace                                                  | Slack node configuration                                                                                  |
| Microsoft Outlook node sends HTML emails so rich formatting is preserved                                                           | Outlook node’s body content type set to HTML                                                             |

---

This document fully details the workflow "Generate Smart Calendly Notifications for Outlook and Slack using GPT-4," enabling advanced users or automation agents to understand, reproduce, and maintain the workflow effectively.