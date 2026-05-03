Send Personalized Follow-Up Emails for Calendly Bookings with GPT-4 and Outlook

https://n8nworkflows.xyz/workflows/send-personalized-follow-up-emails-for-calendly-bookings-with-gpt-4-and-outlook-7306


# Send Personalized Follow-Up Emails for Calendly Bookings with GPT-4 and Outlook

---

### 1. Workflow Overview

This workflow automates sending personalized follow-up emails to users who book meetings via Calendly. Upon a new Calendly booking, it extracts relevant booking details and user responses, uses GPT-4 (via Langchain AI nodes) to generate a customized HTML email tailored to the user's meeting topic, and then sends this email through Microsoft Outlook.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures Calendly booking events (invitee created).
- **1.2 Data Preparation:** Extracts and formats key fields from the Calendly payload for AI processing.
- **1.3 AI Email Generation:** Uses GPT-4 via Langchain nodes to generate a personalized, HTML-formatted email based on extracted data.
- **1.4 Email Dispatch:** Sends the generated email to the invitee using Microsoft Outlook with OAuth2 authentication.
- **1.5 Documentation & Support:** Sticky notes provide setup instructions and contact information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new booking events from Calendly and triggers the workflow whenever a meeting invitee is created.

- **Nodes Involved:**  
  - Calendly Event

- **Node Details:**

  - **Calendly Event**  
    - **Type:** Calendly Trigger node (Webhook-based)  
    - **Configuration:**  
      - Listens specifically for `invitee.created` events.  
      - Uses Calendly API credentials authenticated with a personal access token.  
      - Automatically receives booking details including invitee name, email, meeting start time, and answers to any booking questions.  
    - **Inputs:** Webhook trigger from Calendly.  
    - **Outputs:** Emits detailed JSON payload representing the booking event.  
    - **Potential Failures:**  
      - Authentication failure if API token is invalid or expired.  
      - Webhook misconfiguration causing missed events.  
      - Network timeouts or connectivity issues.  
    - **Version Requirements:** n8n version supporting Calendly Trigger node (v1+).  

#### 2.2 Data Preparation

- **Overview:**  
  Prepares and simplifies the raw Calendly JSON payload by extracting only relevant fields needed for AI processing, ensuring clean and accessible input data.

- **Nodes Involved:**  
  - Edit Fields1

- **Node Details:**

  - **Edit Fields1**  
    - **Type:** Set node  
    - **Configuration:**  
      - Extracts the following fields from the incoming JSON payload and assigns them under `payload`:  
        - `email` (invitee's email)  
        - `questions_and_answers[0].answer` (answer to first booking question)  
        - `name` (invitee name)  
        - `scheduled_event.start_time` (meeting start time)  
      - This simplifies downstream referencing inside expressions.  
    - **Inputs:** Output from Calendly Event node.  
    - **Outputs:** JSON with trimmed and structured data for AI input.  
    - **Potential Failures:**  
      - Missing expected fields if Calendly payload schema changes.  
      - Index out-of-range errors if no questions answered or questions array empty.  
    - **Version Requirements:** n8n Set node v3.4+ for advanced assignment support.

#### 2.3 AI Email Generation

- **Overview:**  
  Uses Langchain AI Agent with GPT-4 as the language model to generate a personalized HTML email based on the invitee's data and meeting details.

- **Nodes Involved:**  
  - AI Agent1  
  - OpenAI Chat Model1  
  - Structured Output Parser1

- **Node Details:**

  - **AI Agent1**  
    - **Type:** Langchain Agent node  
    - **Configuration:**  
      - Input text constructed with invitee name, their answer to the meeting prep question, and the meeting start time.  
      - System message instructs the AI assistant to generate a helpful, elaborated HTML email mentioning how help can be provided, including a placeholder phone number (`111-111-1111`) and the meeting date.  
      - Output expected in JSON format with a key `"email"` containing the HTML content.  
      - Uses defined prompt type with structured output parsing enabled.  
    - **Inputs:** Data from Edit Fields1 and linked to OpenAI Chat Model1 and Structured Output Parser1 for processing and parsing.  
    - **Outputs:** Parsed structured JSON with generated email content.  
    - **Potential Failures:**  
      - OpenAI API errors: quota exceeded, invalid API key, or model unavailability.  
      - Parsing errors if AI output does not match expected JSON schema.  
      - Timeout or latency from the OpenAI API.  
    - **Version Requirements:** Requires Langchain nodes v2+ and OpenAI API access.

  - **OpenAI Chat Model1**  
    - **Type:** Langchain OpenAI Chat model node  
    - **Configuration:**  
      - Model set to GPT-4.1-mini (a GPT-4 variant).  
      - Uses OpenAI API credentials with a valid API key.  
      - No special options configured beyond model selection.  
    - **Inputs:** Receives prompt from AI Agent1.  
    - **Potential Failures:** Same as AI Agent1.

  - **Structured Output Parser1**  
    - **Type:** Langchain Structured Output Parser node  
    - **Configuration:**  
      - Defines expected JSON schema with keys `email` (HTML email string) and `slack` (optional Slack message, unused in this workflow).  
    - **Inputs:** Raw AI generated text from OpenAI Chat Model1.  
    - **Outputs:** Parsed JSON for the AI Agent1 node to consume.  
    - **Potential Failures:** Parsing errors if AI output is malformed or deviates from schema.

#### 2.4 Email Dispatch

- **Overview:**  
  Sends the AI-generated personalized email via Microsoft Outlook to the invitee's email address.

- **Nodes Involved:**  
  - Send a message2

- **Node Details:**

  - **Send a message2**  
    - **Type:** Microsoft Outlook node (Send Email)  
    - **Configuration:**  
      - Subject: Fixed as "Calendly Details".  
      - Body content: Uses the HTML email generated by the AI Agent (`$json.output.email`).  
      - Recipient: The invitee's email extracted earlier (`$('Edit Fields1').item.json.payload.email`).  
      - Body content type set explicitly to HTML for proper rendering.  
      - Uses OAuth2 credentials for Microsoft Outlook authentication.  
    - **Inputs:** Output from AI Agent1 (parsed email content).  
    - **Outputs:** Email sent confirmation message.  
    - **Potential Failures:**  
      - Authentication errors if Outlook OAuth2 token expires or is revoked.  
      - Invalid email addresses causing send failures.  
      - Microsoft API rate limits or service outages.  
    - **Version Requirements:** n8n Microsoft Outlook node v2+ supporting OAuth2.

#### 2.5 Documentation & Support

- **Overview:**  
  Provides accessible instructions and support contact information to users or maintainers of the workflow.

- **Nodes Involved:**  
  - Sticky Note16  
  - Sticky Note

- **Node Details:**

  - **Sticky Note16**  
    - **Type:** Sticky Note  
    - **Content:**  
      - Contact email: robert@ynteractive.com  
      - LinkedIn profile link for further support: https://www.linkedin.com/in/robert-breen-29429625/  
    - **Position:** Positioned separately for visibility.  
    - **No inputs/outputs.**

  - **Sticky Note**  
    - **Type:** Sticky Note  
    - **Content:**  
      - Detailed step-by-step setup instructions covering:  
        - Calendly API credential setup and event configuration.  
        - Microsoft Outlook OAuth2 credential setup.  
        - OpenAI API key addition and AI Agent prompt explanation.  
    - **Position:** Positioned prominently for easy reference.  
    - **No inputs/outputs.**

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                      | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                   |
|--------------------|----------------------------------|------------------------------------|----------------------|----------------------|-----------------------------------------------------------------------------------------------|
| Calendly Event      | Calendly Trigger                 | Receive Calendly booking event     | (Webhook trigger)    | Edit Fields1         |                                                                                               |
| Edit Fields1       | Set                              | Extract and format key fields      | Calendly Event       | AI Agent1            |                                                                                               |
| AI Agent1          | Langchain Agent (GPT-4)           | Generate personalized email content | Edit Fields1, OpenAI Chat Model1, Structured Output Parser1 | Send a message2      |                                                                                               |
| OpenAI Chat Model1 | Langchain OpenAI Chat Model       | GPT-4 language model for AI Agent | AI Agent1            | AI Agent1            |                                                                                               |
| Structured Output Parser1 | Langchain Output Parser           | Parse AI JSON output for email     | OpenAI Chat Model1   | AI Agent1            |                                                                                               |
| Send a message2    | Microsoft Outlook (Send Email)    | Send generated personalized email  | AI Agent1            |                      |                                                                                               |
| Sticky Note16      | Sticky Note                      | Support contact info                |                      |                      | 📬 Need Help or Want to Customize This? 📧 robert@ynteractive.com 🔗 https://www.linkedin.com/in/robert-breen-29429625/ |
| Sticky Note        | Sticky Note                      | Setup instructions                 |                      |                      | ⚙️ Step-by-Step Setup Instructions: Calendly API, Outlook OAuth2, OpenAI API setup detailed.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Calendly Event Trigger Node**  
   - Type: Calendly Trigger  
   - Configure to listen for event: `invitee.created`  
   - Add Calendly API credential with your personal access token.  
   - Save webhook URL and ensure it is registered in Calendly webhook settings.

2. **Create Set Node (Edit Fields1)**  
   - Type: Set  
   - Add assignments to extract and assign the following JSON fields under `payload`:  
     - `email`: `{{$json.payload.email}}`  
     - `questions_and_answers[0].answer`: `{{$json.payload.questions_and_answers[0].answer}}`  
     - `name`: `{{$json.payload.name}}`  
     - `scheduled_event.start_time`: `{{$json.payload.scheduled_event.start_time}}`  
   - Connect output of Calendly Event node to this Set node.

3. **Create Langchain AI Agent Node (AI Agent1)**  
   - Type: Langchain Agent  
   - Configure the input text expression as:  
     ```
     Name: {{$json.payload.name}} About: {{$json.payload.questions_and_answers[0].answer}} Start Time: {{$json.payload.scheduled_event.start_time}}
     ```  
   - Set system message:  
     ```
     You are a helpful assistant. Write an email to the user. Elaborate on how we can help them with their topic and give my fake phone number for now 111-111-1111. Write the email in html. also put the meeting date in there. 

     output like this. 

     {
       "email": "html email"
     }
     ```  
   - Enable structured output parsing.  
   - Connect the output from Edit Fields1 node.

4. **Create Langchain OpenAI Chat Model Node (OpenAI Chat Model1)**  
   - Type: Langchain OpenAI Chat Model  
   - Select GPT-4 (e.g., `gpt-4.1-mini`) as the model.  
   - Add OpenAI API credentials with your API key.  
   - Connect this node to the AI Agent1 as the language model input.

5. **Create Langchain Structured Output Parser Node (Structured Output Parser1)**  
   - Type: Langchain Structured Output Parser  
   - Provide JSON schema example:  
     ```json
     {
       "email": "html email",
       "slack": "Slack Message"
     }
     ```  
   - Connect this node to the OpenAI Chat Model1 as the output parser.

6. **Connect Langchain nodes properly:**  
   - AI Agent1 connects to OpenAI Chat Model1 (ai_languageModel) and Structured Output Parser1 (ai_outputParser).  
   - Structured Output Parser1 and OpenAI Chat Model1 feed back into AI Agent1.

7. **Create Microsoft Outlook Node (Send a message2)**  
   - Type: Microsoft Outlook (Send Email)  
   - Set subject: `"Calendly Details"`  
   - Set body content: use expression referencing AI Agent1 output email: `{{$json.output.email}}`  
   - Set recipient: expression referencing invitee email from Edit Fields1: `{{$('Edit Fields1').item.json.payload.email}}`  
   - Set body content type to `HTML`.  
   - Add Microsoft Outlook OAuth2 credentials.  
   - Connect output from AI Agent1 to this node.

8. **Add Sticky Notes for Documentation**  
   - Create Sticky Note with contact info: email and LinkedIn URL.  
   - Create Sticky Note with detailed step-by-step setup instructions for Calendly, Outlook, and OpenAI API setup.

9. **Test the workflow**  
   - Trigger a test booking in Calendly to ensure the webhook fires.  
   - Confirm email generation and delivery through Outlook.  
   - Check logs for any errors or missing data.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| For help or customization, contact Robert Breen at robert@ynteractive.com or visit LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Support contact information provided in Sticky Note16.                                                |
| Setup instructions include detailed guidance for Calendly API credential configuration, Microsoft Outlook OAuth2 setup, and OpenAI API key integration. | Refer to Sticky Note with step-by-step instructions within the workflow canvas.                        |
| The workflow uses placeholder phone number `111-111-1111` in generated emails; update as needed.  | Part of AI Agent system prompt.                                                                        |
| Ensure correct OAuth2 scopes are granted for Microsoft Outlook to send emails on behalf of the user. | Credential setup pre-requisite.                                                                        |
| Langchain nodes require n8n with Langchain integration and compatible OpenAI API access.           | Verify n8n version and installed packages.                                                            |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.