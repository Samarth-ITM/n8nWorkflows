Automated Patient Response System with GPT-3.5 and Google Sheets for Dental Clinics

https://n8nworkflows.xyz/workflows/automated-patient-response-system-with-gpt-3-5-and-google-sheets-for-dental-clinics-7971


# Automated Patient Response System with GPT-3.5 and Google Sheets for Dental Clinics

### 1. Workflow Overview

This workflow automates the patient lead engagement process for dental clinics by integrating Google Sheets, OpenAI GPT-3.5, and Gmail. It targets dental clinics receiving new patient inquiries via Google Forms linked to a Google Sheet. The workflow listens for new form submissions, generates a personalized, friendly welcome email using GPT-3.5 tailored to the patient's dental needs and interests, and sends immediate notifications both to the patient and to the clinic's email for follow-up.

Logical blocks include:

- **1.1 Input Reception:** Detects new patient lead submissions from Google Sheets (populated by Google Forms).
- **1.2 Wait & Data Preparation:** Brief pause after trigger to ensure data stability.
- **1.3 AI Processing:** Uses LangChain n8n nodes with GPT-3.5 to generate a custom HTML email based on patient form inputs.
- **1.4 Email Dispatch:** Sends the personalized welcome email to the patient and a notification email to the clinic's internal address.
- **1.5 Memory & Thinking Tools:** Maintains conversational context and intermediate processing to enhance AI output quality.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Detects new rows added to a specific Google Sheet (populated by patient form submissions). Triggers the workflow on each new patient lead submission.

**Nodes Involved:**  
- Google Sheets Trigger  
- Sticky Note1 (contextual note)

**Node Details:**

- **Google Sheets Trigger**  
  - Type: Trigger node for Google Sheets  
  - Configuration: Watches for the event "rowAdded" on the "Form responses 1" sheet inside the designated Google Sheet document. Polls every minute.  
  - Credentials: Uses OAuth2 for Google Sheets API access.  
  - Input/Output: No input; outputs the new row data as JSON for downstream nodes.  
  - Edge Cases:  
    - Google API rate limits or connectivity issues.  
    - Delay in Google Sheets sync with Forms can cause missed triggers if polling frequency is too low.  
  - Sticky Note1 provides the Google Form URL linked to this sheet: https://docs.google.com/forms/u/0/d/e/1FAIpQLSevL3LaoKWTLuekELB0Mrp6k5Vh5WZockkdv0Lefn1_YjtXg/formResponse  

---

#### 2.2 Wait & Data Preparation

**Overview:**  
Introduces a short delay after trigger activation to ensure data consistency before processing.

**Nodes Involved:**  
- Wait  
- Sticky Note2 (contextual note about data storage)

**Node Details:**

- **Wait**  
  - Type: Wait node  
  - Configuration: Default wait with no additional parameters (uses default delay, likely minimal or zero delay).  
  - Input: Output from Google Sheets Trigger.  
  - Output: Passes the same data downstream after waiting.  
  - Edge Cases: Unlikely to fail but if delay is zero or too short, race conditions may occur.  

- **Sticky Note2** reminds that all leads are saved in the Google Sheet for clinic records: https://docs.google.com/spreadsheets/d/1RLC2D_t-NYuhbKD8WV-EPh_INH7idEVhslXNbP2-jSE/edit?resourcekey=&gid=951804608#gid=951804608

---

#### 2.3 AI Processing

**Overview:**  
Constructs a detailed prompt from form data and uses GPT-3.5 via LangChain to generate a warm, personalized HTML welcome email for the patient.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  
- Think  
- AI Agent  
- Sticky Note3 (AI Agent description)

**Node Details:**

- **Simple Memory**  
  - Type: Memory buffer node from LangChain  
  - Role: Stores conversation context keyed by patient's email to maintain session context if needed.  
  - Configuration: Session key set to patient's email address from the form JSON.  
  - Input: Receives data from Wait.  
  - Output: Provides memory context to AI Agent.  
  - Edge Cases: Failure if email is missing or malformed; session context may reset if email changes.

- **OpenAI Chat Model**  
  - Type: Language model node (GPT-3.5-turbo)  
  - Role: Provides the AI model that generates text based on prompts.  
  - Configuration: Uses GPT-3.5-turbo with default options. Credentials use OpenAI API key.  
  - Input: Connected as the language model for AI Agent.  
  - Edge Cases: API key issues, model rate limits, latency, or timeout.

- **Think**  
  - Type: LangChain Thinking tool  
  - Role: Intermediate step to process or refine AI outputs before final generation.  
  - Input/Output: Linked to AI Agent as an AI tool.  
  - Edge Cases: Processing errors or timeouts.

- **AI Agent**  
  - Type: LangChain agent node orchestrating AI conversation  
  - Role: Prepares patient-specific prompt including form fields (Full Name, Email, Treatment Interest, Dental Concern, Contact Method, Contact Time), and instructs GPT to generate a personalized HTML email with subject, greeting, paragraphs, emojis, and a call-to-action button linking to WhatsApp booking.  
  - Configuration: Includes a detailed system message guiding tone, content, and HTML formatting.  
  - Key Expressions: Uses JSON expressions to populate prompt dynamically from Google Sheets trigger data.  
  - Input: Receives memory context and Think node output.  
  - Output: Produces full HTML email content.  
  - Edge Cases: Expression errors if form data fields are missing or misnamed; AI output may be malformed HTML; API or integration errors.  
  - Sticky Note3 highlights the role: "🤖 Generate friendly, personalized welcome message."

---

#### 2.4 Email Dispatch

**Overview:**  
Sends the personalized welcome email to the patient and simultaneously notifies the clinic team with lead details.

**Nodes Involved:**  
- Send a message  
- Send a message1  
- Sticky Note4 (Patient Email)  
- Sticky Note5 (Dentist Notification)

**Node Details:**

- **Send a message**  
  - Type: Gmail node for sending email  
  - Role: Sends the AI-generated personalized welcome email to the patient email from form data.  
  - Configuration:  
    - `sendTo` dynamically set to patient email from Google Sheets.  
    - `message` set to AI Agent output (HTML email).  
    - Subject: "Welcome to SmileBright Dental" (static).  
  - Credentials: Gmail OAuth2 account.  
  - Edge Cases: Gmail API limits, invalid patient email, SMTP errors.

- **Send a message1**  
  - Type: Gmail node  
  - Role: Sends notification email to the dental clinic's internal email (n8nagency.online@gmail.com) with patient lead details for follow-up.  
  - Configuration:  
    - `sendTo`: fixed clinic email.  
    - `message`: text summary including patient name, email, phone, treatment interest, dental concern, best contact method and time.  
    - Subject: "New Lead {Patient Full Name}".  
  - Credentials: Same Gmail OAuth2 account.  
  - Edge Cases: SMTP or API errors, missing patient data fields.

- **Sticky Note4** notes the patient notification purpose: "**SEND** 📧 personalized welcome email, appointment notification, immediately."

- **Sticky Note5** notes the dentist notification purpose: "**SEND** 📧 personalized welcome email, appointment notification, immediately."

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                       | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                      |
|---------------------|----------------------------------|------------------------------------|------------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger| n8n-nodes-base.googleSheetsTrigger| Detect new patient lead submissions | None                   | Wait                  | ##  Trigger ⚡ Fires when a new patient submits the form. form is: https://docs.google.com/forms/...            |
| Wait                | n8n-nodes-base.wait               | Delay to ensure data stability      | Google Sheets Trigger  | AI Agent              | ## 📊 Save every lead for clinic records. in google sheet https://docs.google.com/spreadsheets/...               |
| Simple Memory       | LangChain memoryBufferWindow       | Store session context by email     | (Implicit via AI Agent) | AI Agent              |                                                                                                                 |
| OpenAI Chat Model   | LangChain lmChatOpenAi             | Provide GPT-3.5 model for AI       | AI Agent (languageModel)| AI Agent              |                                                                                                                 |
| Think               | LangChain toolThink                | Intermediate AI processing         | AI Agent (ai_tool)     | AI Agent              |                                                                                                                 |
| AI Agent            | LangChain agent                   | Generate personalized HTML email   | Wait, Simple Memory, Think, OpenAI Chat Model | Send a message1 | ## AI AGENT 🤖 Generate friendly, personalized welcome message.                                                |
| Send a message1     | n8n-nodes-base.gmail               | Notify clinic with lead details    | AI Agent               | Send a message         | ## Dentist Notification **SEND** 📧 personalized welcome email ,appointment notification,immediately.           |
| Send a message      | n8n-nodes-base.gmail               | Send personalized email to patient | Send a message1         | None                  | ## Patient Notification **SEND** 📧 personalized welcome email ,appointment notification,immediately.           |
| Sticky Note1        | n8n-nodes-base.stickyNote          | Context - Trigger form URL          | None                   | None                  | ##  Trigger ⚡ Fires when a new patient submits the form. form is: https://docs.google.com/forms/...            |
| Sticky Note2        | n8n-nodes-base.stickyNote          | Context - Data saved in Google Sheets| None                   | None                  | ## 📊 Save every lead for clinic records. in google sheet https://docs.google.com/spreadsheets/...               |
| Sticky Note3        | n8n-nodes-base.stickyNote          | Context - AI Agent role             | None                   | None                  | ## AI AGENT 🤖 Generate friendly, personalized welcome message.                                                |
| Sticky Note4        | n8n-nodes-base.stickyNote          | Context - Patient email notification| None                   | None                  | ## Patient Notification **SEND** 📧 personalized welcome email ,appointment notification,immediately.           |
| Sticky Note5        | n8n-nodes-base.stickyNote          | Context - Dentist email notification| None                   | None                  | ## Dentist Notification **SEND** 📧 personalized welcome email ,appointment notification,immediately.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node**  
   - Node type: Google Sheets Trigger  
   - Set event to "rowAdded" on the sheet named "Form responses 1" in your target Google Sheet document.  
   - Configure OAuth2 credentials for Google Sheets API access.  
   - Set polling interval to every minute.

2. **Add Wait node**  
   - Node type: Wait (default configuration).  
   - Connect Google Sheets Trigger output to Wait node input.

3. **Set up Simple Memory node (LangChain memoryBufferWindow)**  
   - Node type: memoryBufferWindow  
   - Under parameters, set `sessionKey` to `={{ $('Google Sheets Trigger').item.json['Email address'] }}`  
   - Set `sessionIdType` to "customKey".  
   - Connect Wait node output to Simple Memory input.

4. **Add OpenAI Chat Model node**  
   - Node type: lmChatOpenAi  
   - Select model `gpt-3.5-turbo`.  
   - Configure with OpenAI API credentials.  
   - No additional parameters needed.  

5. **Add Think node (LangChain toolThink)**  
   - Node type: toolThink  
   - No parameters required.  
   - Connect OpenAI Chat Model output to Think node input.

6. **Configure AI Agent node**  
   - Node type: LangChain agent  
   - Parameters:  
     - Text prompt (use expressions to include patient form fields):  
       ```
       Patient Details:
       - Full Name: {{$json["Full Name"]}}
       - Email: {{$json["Email"]}}
       - Treatment Interest: {{$json[' What are you most interested in?']}}
       - Dental Concern: {{$json["What’s your main dental concern right now? (e.g., yellow teeth, missing tooth, crooked smile)"]}}
       - Contact Method: {{$json[' What’s the best way to reach you?']}}
       - Contact Time: {{$json["What’s the best time to reach you?"]}}
       ```
     - System message (set as per the workflow):  
       ```
       You are a friendly and professional dental clinic assistant for SmileBright Dental. 
       Your task is to write a short, warm, and engaging welcome email based on the patient's form inputs. 
       Tailor the email to their treatment choice, mentioning it positively (e.g., Teeth Whitening → ✨ brighter smile, Invisalign → 😁 confident smile, Dental Implants → 🦷 restored smile). 
       Use their main dental concern for personalization. 
       Encourage them to book an appointment and include their preferred contact method and time. 
       Keep the tone enthusiastic, professional, and concise.
       
       ⚠️ Important: Return the output as a complete **HTML email** with:
       - A subject line in <title> at the top (for the email subject)
       - An <h1> or <h2> for greeting
       - Paragraphs <p> for the body
       - Emojis to make it friendly
       - A clear clickable call-to-action button (<a href="https://wa.me/..." style="...">Book Appointment</a>)
       
       Do not return JSON or plain text. Only valid HTML.
       ```
   - Connect Simple Memory output as `ai_memory` input.  
   - Connect Think node as `ai_tool` input.  
   - Connect OpenAI Chat Model as `ai_languageModel` input.  
   - Connect Wait node output as main input.

7. **Add Send a message1 node (Clinic notification email)**  
   - Node type: Gmail  
   - Configure OAuth2 Gmail credentials.  
   - Set `sendTo` to clinic email (e.g., n8nagency.online@gmail.com).  
   - Compose a plain text message summarizing lead details using expressions from Google Sheets trigger JSON fields.  
   - Subject: "New Lead {Patient Full Name}" dynamically populated.  
   - Connect AI Agent output to this node.

8. **Add Send a message node (Patient email)**  
   - Node type: Gmail  
   - Configure OAuth2 Gmail credentials (can be same as above).  
   - Set `sendTo` dynamically to patient email from Google Sheets trigger JSON.  
   - Set `message` to the HTML output from AI Agent.  
   - Subject: "Welcome to SmileBright Dental" (static).  
   - Connect Send a message1 output to this node.

9. **Verify connections:**  
   - Google Sheets Trigger -> Wait -> Simple Memory -> AI Agent -> Send a message1 -> Send a message  
   - Additionally, AI Agent receives inputs from OpenAI Chat Model, Think, and Simple Memory nodes.

10. **Set up credentials:**  
    - Google Sheets Trigger: OAuth2 credentials with Google Sheets API access.  
    - OpenAI Chat Model: OpenAI API key credentials.  
    - Send a message nodes: Gmail OAuth2 credentials for sending emails.

11. **Test the workflow:**  
    - Submit a test patient form entry to Google Forms linked to the Google Sheet.  
    - Confirm trigger activation, AI-generated personalized email sent to patient, and notification email sent to clinic.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Dental clinics lose many new leads because generic thank-you emails lack engagement; this workflow solves it by sending personalized, warm, immediate emails. | Sticky Note on workflow start explaining problem context.                                                |
| Patient form URL link: https://docs.google.com/forms/u/0/d/e/1FAIpQLSevL3LaoKWTLuekELB0Mrp6k5Vh5WZockkdv0Lefn1_YjtXg/formResponse | Linked in Sticky Note1 for form source.                                                                  |
| Google Sheet URL storing leads: https://docs.google.com/spreadsheets/d/1RLC2D_t-NYuhbKD8WV-EPh_INH7idEVhslXNbP2-jSE/edit#gid=951804608 | Sticky Note2 reference to data storage for records.                                                      |
| AI Agent generates HTML emails with emojis and call-to-action buttons styled for WhatsApp booking. | Sticky Note3 detailing AI Agent role and output format.                                                  |
| Immediate email notifications to patient and clinic ensure quick engagement and follow-up.        | Sticky Notes4 and 5 emphasize the sending of emails to respective recipients.                            |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.