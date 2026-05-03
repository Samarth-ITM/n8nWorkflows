Post-Surgery Patient Triage & Follow-up System with Gemini AI, Telegram & Google Suite

https://n8nworkflows.xyz/workflows/post-surgery-patient-triage---follow-up-system-with-gemini-ai--telegram---google-suite-11649


# Post-Surgery Patient Triage & Follow-up System with Gemini AI, Telegram & Google Suite

---

## 1. Workflow Overview

This workflow is an AI-powered post-surgery patient triage and follow-up system integrating Google Sheets, Telegram, Google Calendar, Gmail, and Google Gemini AI. It automates daily patient follow-ups, message triage, empathetic responses, appointment scheduling, and urgent notifications to medical staff.

**Target Use Cases:**  
- Automated daily check-ins with post-surgery patients within their follow-up period  
- AI-driven analysis of patient messages to classify symptom intensity (low, moderate, high)  
- Tailored, empathetic responses sent via Telegram according to message intensity  
- Scheduling follow-up appointments for moderate concerns  
- Immediate doctor notifications and automated calls for high-intensity concerns

**Logical Blocks:**

- **1.1 Daily Patient Follow-up Trigger & Retrieval:** Scheduled trigger to retrieve patient data from Google Sheets and filter for active follow-up cases.
- **1.2 Automated Patient Messaging:** Sends personalized Telegram messages to patients to check on their recovery.
- **2.1 Incoming Patient Message Handling:** Receives patient messages via Telegram, extracts and sets relevant fields.
- **2.2 AI Triage of Patient Messages:** Uses AI to summarize and classify message intensity, deciding response urgency.
- **3.1 Routing Based on Intensity:** Switch node routes to different response paths (low, moderate, high).
- **3.2 Low-Intensity Response:** Sends a reassuring Telegram message for minor concerns.
- **3.3 Moderate-Intensity Response:** Sends empathetic Telegram response and schedules follow-up appointment.
- **3.4 High-Intensity Response:** Sends urgent Telegram response, notifies doctor via email, and initiates automated call.
- **4. AI-Driven Communications:** Multiple AI agents using Google Gemini model generate summaries, triage classifications, empathetic replies, and formal emails.
- **5. Appointment Scheduling & Notifications:** Creates calendar events and sends Telegram reminders for follow-ups.

---

## 2. Block-by-Block Analysis

### 1.1 Daily Patient Follow-up Trigger & Retrieval

**Overview:**  
Triggers the workflow daily at 9 AM to retrieve patient data from Google Sheets and filters patients who are currently in their follow-up period based on discharge date and follow-up duration.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  
- Code (filter)  
- Send a text message (initial follow-up message)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule (time-based start)  
  - Configuration: Fires daily at 9:00 AM  
  - Inputs: None (trigger node)  
  - Outputs: Starts data retrieval flow  
  - Potential Failures: Scheduling misconfiguration, n8n instance downtime

- **Get row(s) in sheet**  
  - Type: Google Sheets (read rows)  
  - Configuration: Reads all rows from Sheet1 in specified Google Sheet (patient list)  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Patient records as JSON  
  - Potential Failures: Authentication errors, spreadsheet access issues, empty or malformed data

- **Code**  
  - Type: Code (JavaScript)  
  - Configuration: Filters patients who have been discharged (start_date in past) and are still within their follow-up duration  
  - Key Expression: Calculates days since discharge and compares with Follow_up_duration(days) column  
  - Inputs: Patient data array  
  - Outputs: Filtered patient data to message node  
  - Edge Cases: Invalid or missing dates, non-numeric follow-up days, timezone issues

- **Send a text message**  
  - Type: Telegram (send message)  
  - Configuration: Sends personalized follow-up message to patient’s Telegram chat ID (hardcoded chatId in JSON is 5443133930 — likely a placeholder)  
  - Message uses expression to address patient by name from filtered data  
  - Credentials: Telegram API (OAuth)  
  - Inputs: Filtered patients from Code node  
  - Outputs: None (end of this branch)  
  - Edge Cases: Invalid chatId, Telegram API rate limits, message formatting errors

---

### 1.2 Automated Patient Messaging

This block is combined with 1.1: the Telegram message sent daily to patients in follow-up.

---

### 2.1 Incoming Patient Message Handling

**Overview:**  
Listens for incoming Telegram messages from patients, extracts essential fields (first name, message text, chat id) for downstream AI processing.

**Nodes Involved:**  
- Telegram Trigger  
- Edit Fields (Set node)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger (Webhook)  
  - Configuration: Listens for incoming Telegram messages (updates: message)  
  - Credentials: Telegram API  
  - Inputs: Incoming Telegram messages  
  - Outputs: Raw Telegram message JSON

- **Edit Fields**  
  - Type: Set  
  - Configuration: Extracts and sets fields:  
    - first_name from message.chat.first_name  
    - message.text from message.text  
    - message.chat.id for reply targeting  
  - Inputs: Telegram Trigger output  
  - Outputs: Structured JSON for AI agents  
  - Edge Cases: Missing or malformed Telegram message fields

---

### 2.2 AI Triage of Patient Messages

**Overview:**  
Uses a chain of AI agents powered by Google Gemini to summarize patient messages, classify symptom intensity, and generate appropriate empathetic responses.

**Nodes Involved:**  
- AI Agent (summary)  
- Google Gemini Chat Model (supporting AI calls)  
- AI Agent4 (triage classification)  
- Google Gemini Chat Model4 (AI model for triage)  
- Code1 (parse AI classification output)  
- If (check status)  
- Switch (route by intensity)

**Node Details:**

- **AI Agent**  
  - Type: Langchain Agent  
  - Configuration: Summarizes patient messages into clear, medically useful summaries without fabricating information.  
  - Uses Google Gemini Chat Model as LLM backend  
  - Inputs: Edited patient message data  
  - Outputs: Summary text to AI Agent4 for triage

- **AI Agent4**  
  - Type: Langchain Agent  
  - Configuration: Classifies summarized message into status ("read" or "ignore") and intensity (low, moderate, high)  
  - Output format: "status: read\nintensity: moderate" or "status: ignore"  
  - Inputs: Summary text from AI Agent  
  - Outputs: Classification text to Code1

- **Code1**  
  - Type: Code  
  - Configuration: Parses AI output string into JSON object with keys "status" and "intensity"  
  - Inputs: AI Agent4 output text  
  - Outputs: Parsed JSON for If node

- **If**  
  - Type: If  
  - Configuration: Checks if status equals "read" to decide if message needs response routing  
  - Inputs: Code1 parsed output  
  - Outputs: True branch to Switch node, False branch to feedback message

- **Switch**  
  - Type: Switch  
  - Configuration: Routes messages by intensity: high, moderate, low  
  - Inputs: If true branch output  
  - Outputs: Routes to AI Agent3 (high), AI Agent2 (moderate), AI Agent1 (low)

- **Google Gemini Chat Model and related**  
  - These nodes serve as LLM connectors for respective AI Agents, handling calls to Google Gemini (PaLM) API  
  - Credentials: Google Palm API key required  
  - Potential Failures: API limits, credential expiry, network errors

---

### 3.1 Routing Based on Intensity

**Overview:**  
Switch node routes message processing and response generation according to triage results.

**Nodes Involved:**  
- Switch  
- AI Agent1 (low)  
- AI Agent2 (moderate)  
- AI Agent3 (high)  
- Corresponding Google Gemini Chat Models (1,2,3)  
- Get row(s) in sheet in Google Sheets (1,2,3) for patient data enrichment

**Node Details:**

- **AI Agent1 (Low Intensity Response)**  
  - Generates a warm, brief, reassuring Telegram reply for low-intensity concerns.

- **AI Agent2 (Moderate Intensity Response)**  
  - Generates empathetic response that acknowledges mild to moderate symptoms, reassures, and encourages follow-up.

- **AI Agent3 (High Intensity Response)**  
  - Generates urgent, caring response indicating prioritization and immediate medical attention.

- **Get row(s) in sheet in Google Sheets1,2,3**  
  - Fetches patient details from Google Sheets based on incoming message context to enrich AI responses.

- Each AI Agent uses Google Gemini Chat Model nodes for LLM execution.

---

### 3.2 Low-Intensity Response

**Overview:**  
Sends a friendly Telegram message to patients reporting minor symptoms or no concerns.

**Nodes Involved:**  
- AI Agent1  
- Google Gemini Chat Model1  
- Send a text message4 (Telegram send)

**Node Details:**

- **Send a text message4**  
  - Sends AI-generated empathetic reply to patient chat ID  
  - Credentials: Telegram API  
  - Edge Cases: Invalid chat ID, Telegram delivery failure

---

### 3.3 Moderate-Intensity Response

**Overview:**  
Sends an empathetic Telegram reply and schedules a follow-up appointment for patients with moderate symptoms.

**Nodes Involved:**  
- AI Agent2  
- Google Gemini Chat Model2  
- Send a text message2 (Telegram send)  
- AI Agent6 (appointment scheduling and notification)  
- Google Gemini Chat Model6  
- Create an event in Google Calendar  
- Send a text message in Telegram (appointment notification)

**Node Details:**

- **AI Agent6**  
  - Generates appointment details (date/time) and a Telegram message informing the patient about the scheduled follow-up.

- **Create an event in Google Calendar**  
  - Creates appointment event in specified Google Calendar account using AI-generated start/end times.

- **Send a text message in Telegram**  
  - Notifies patient of scheduled appointment with friendly message.

- **Potential Failures:** Calendar API errors, appointment data format issues, Telegram message failures

---

### 3.4 High-Intensity Response

**Overview:**  
Sends urgent Telegram reply, notifies the doctor via email, and initiates an automated phone call for serious patient concerns.

**Nodes Involved:**  
- AI Agent3  
- Google Gemini Chat Model3  
- Send a text message3 (Telegram send)  
- AI Agent5 (generate doctor email content)  
- Google Gemini Chat Model5  
- Get row(s) in sheet in Google Sheets4 (patient data for email)  
- Send a message in Gmail (email to doctor)  
- HTTP Request (calls VAPI.ai for automated phone call)

**Node Details:**

- **AI Agent5**  
  - Creates a formal, concise email to the patient's doctor describing the urgency and nature of the concern.

- **Send a message in Gmail**  
  - Uses Gmail OAuth2 credentials to send the email to the doctor.

- **HTTP Request**  
  - Sends API call to VAPI.ai to initiate a phone call alerting the doctor or care team.

- **Potential Failures:** Email sending errors, API call failures, missing or incorrect doctor email, authentication issues

---

### 4. AI-Driven Communications

**Overview:**  
Several AI Agent nodes leverage Google Gemini LLM to generate message summaries, triage classifications, empathetic replies, appointment details, and professional emails.

**Nodes Involved:**  
- Multiple AI Agent nodes (AI Agent, AI Agent1-6)  
- Corresponding Google Gemini Chat Model nodes

**Node Details:**

- Each AI Agent node includes specialized prompt templates tailored to medical assistant roles, emphasizing no fabrication, clarity, empathy, and professionalism.

- Google Gemini Chat Model nodes connect to Google PaLM API using configured credentials.

- Proper error handling for API limits and response parsing is important.

---

### 5. Appointment Scheduling & Notifications

**Overview:**  
Schedules follow-up appointments in Google Calendar and informs patients via Telegram messages.

**Nodes Involved:**  
- AI Agent6  
- Google Gemini Chat Model6  
- Create an event in Google Calendar  
- Send a text message in Telegram

---

## 3. Summary Table

| Node Name                           | Node Type                   | Functional Role                            | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                     |
|-----------------------------------|-----------------------------|-------------------------------------------|--------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | Schedule Trigger             | Daily trigger to start follow-up retrieval| None                           | Get row(s) in sheet             | ## 1. Pull patients details to check if follow up is needed                                                      |
| Get row(s) in sheet              | Google Sheets                | Retrieves patient data from Google Sheet  | Schedule Trigger               | Code                           | ## 1. Pull patients details to check if follow up is needed                                                      |
| Code                            | Code                        | Filters patients currently in follow-up   | Get row(s) in sheet           | Send a text message             | ## 1. Pull patients details to check if follow up is needed                                                      |
| Send a text message             | Telegram                    | Sends daily follow-up messages to patients| Code                          | None                          | ## 1. Pull patients details to check if follow up is needed                                                      |
| Telegram Trigger                | Telegram Trigger            | Listens for incoming patient messages     | None                         | Edit Fields                    | ## 2. review patient messages to clasify the intensity                                                           |
| Edit Fields                    | Set                         | Extracts first_name, message text, chat id| Telegram Trigger             | AI Agent                      | ## 2. review patient messages to clasify the intensity                                                           |
| AI Agent                      | Langchain Agent             | Summarizes patient messages                | Edit Fields                  | AI Agent4                     | ## 2. review patient messages to clasify the intensity                                                           |
| Google Gemini Chat Model       | Langchain LLM Model         | Backend LLM for AI Agent                    | AI Agent                     | AI Agent                      | ## 2. review patient messages to clasify the intensity                                                           |
| AI Agent4                     | Langchain Agent             | Classifies message status and intensity   | AI Agent                     | Code1                         | ## 2. review patient messages to clasify the intensity                                                           |
| Google Gemini Chat Model4      | Langchain LLM Model         | Backend LLM for AI Agent4                   | AI Agent4                    | AI Agent4                     | ## 2. review patient messages to clasify the intensity                                                           |
| Code1                         | Code                        | Parses AI classification output            | AI Agent4                    | If                           | ## 2. review patient messages to clasify the intensity                                                           |
| If                            | If                          | Checks if message status is "read"         | Code1                        | Switch / feedback msg          | ## 3. route response                                                                                              |
| Switch                        | Switch                      | Routes messages by intensity (high/moderate/low) | If                      | AI Agent3 / AI Agent2 / AI Agent1 | ## 3. route response                                                                                              |
| AI Agent1                     | Langchain Agent             | Generates low intensity empathetic reply  | Switch (low)                 | Send a text message4           | ## 4. Low intensity reply                                                                                          |
| Google Gemini Chat Model1      | Langchain LLM Model         | Backend LLM for AI Agent1                   | AI Agent1                    | AI Agent1                     | ## 4. Low intensity reply                                                                                          |
| Send a text message4          | Telegram                    | Sends low intensity reply to patient       | AI Agent1                    | None                         | ## 4. Low intensity reply                                                                                          |
| AI Agent2                     | Langchain Agent             | Generates moderate intensity empathetic reply | Switch (moderate)           | Send a text message2 / AI Agent6 | ## 5.  Resasure assure patient and schedule an appointment                                                        |
| Google Gemini Chat Model2      | Langchain LLM Model         | Backend LLM for AI Agent2                   | AI Agent2                    | AI Agent2                     | ## 5.  Resasure assure patient and schedule an appointment                                                        |
| Send a text message2          | Telegram                    | Sends moderate intensity reply to patient | AI Agent2                    | AI Agent6                    | ## 5.  Resasure assure patient and schedule an appointment                                                        |
| AI Agent6                     | Langchain Agent             | Generates appointment details and notification message | Send a text message2      | Google Gemini Chat Model6 / Create an event / Send a text message in Telegram | ## 5.  Resasure assure patient and schedule an appointment                                                        |
| Google Gemini Chat Model6      | Langchain LLM Model         | Backend LLM for AI Agent6                   | AI Agent6                    | AI Agent6                     | ## 5.  Resasure assure patient and schedule an appointment                                                        |
| Create an event in Google Calendar | Google Calendar           | Creates follow-up appointment event        | AI Agent6                    | Send a text message in Telegram| ## 5.  Resasure assure patient and schedule an appointment                                                        |
| Send a text message in Telegram| Telegram                    | Sends appointment notification to patient  | Create an event              | None                         | ## 5.  Resasure assure patient and schedule an appointment                                                        |
| AI Agent3                     | Langchain Agent             | Generates high intensity urgent reply      | Switch (high)                | Send a text message3 / AI Agent5 | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| Google Gemini Chat Model3      | Langchain LLM Model         | Backend LLM for AI Agent3                   | AI Agent3                    | AI Agent3                     | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| Send a text message3          | Telegram                    | Sends urgent reply to patient               | AI Agent3                    | AI Agent5                    | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| AI Agent5                     | Langchain Agent             | Generates formal email to doctor            | Send a text message3         | Google Gemini Chat Model5 / Get row(s) in sheet in Google Sheets4 / Send a message in Gmail / HTTP Request | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| Google Gemini Chat Model5      | Langchain LLM Model         | Backend LLM for AI Agent5                   | AI Agent5                    | AI Agent5                     | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| Get row(s) in sheet in Google Sheets4 | Google Sheets           | Retrieves patient data for doctor email     | AI Agent5                    | Send a message in Gmail       | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| Send a message in Gmail       | Gmail                       | Sends email notification to doctor          | Get row(s) in sheet in Google Sheets4 | None                         | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |
| HTTP Request                 | HTTP Request                | Calls VAPI.ai API to initiate automated call | Send a message in Gmail      | AI Agent5                    | ## 6. Initiate call, inform doctor and scheduledappointment for high intensity                                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configure to run daily at 9:00 AM

2. **Add Google Sheets Node ("Get row(s) in sheet")**  
   - Type: Google Sheets  
   - Configure to read all rows from the patient list spreadsheet (Sheet1)  
   - Connect Schedule Trigger output to this node  
   - Set Google Sheets OAuth2 credentials

3. **Add Code Node ("Code") to filter follow-up patients**  
   - Add JavaScript code to filter patients discharged before today and within follow-up duration:  
     ```js
     const today = new Date();
     return items.filter(item => {
       const data = item.json;
       const dischargeDate = new Date(data.start_date);
       const followUpDays = Number(data["Follow_up_duration(days)"]);
       const daysSinceDischarge = Math.floor((today - dischargeDate) / (1000 * 60 * 60 * 24));
       return daysSinceDischarge > 0 && daysSinceDischarge <= followUpDays;
     });
     ```  
   - Connect "Get row(s) in sheet" output to this

4. **Add Telegram Node ("Send a text message")**  
   - Type: Telegram  
   - Use Telegram OAuth credentials  
   - Text message with expression:  
     ```
     Hello {{ $('Code').item.json.Name }}

     We hope your recovery is going smoothly and that you're feeling a little better today. 💛

     If you're feeling any pain, discomfort, or have *any concerns at all*, please don’t hesitate to reach out.  
     We're here to support you — just reply to this message, and we’ll get back to you as soon as possible. 

     you are free to tell us what are you feeling!

     Take it easy, rest well, and remember healing takes time. You've got this. 💪

     – Your Care Team
     ```  
   - Connect Code output to this node  
   - Set chatId appropriately (dynamic from patient data or predefined)

5. **Add Telegram Trigger to receive incoming messages**  
   - Type: Telegram Trigger  
   - Connect webhook to receive patient messages  
   - Use Telegram credentials

6. **Add Set Node ("Edit Fields")**  
   - Extract first_name, message.text, message.chat.id from Telegram trigger output fields using expressions  
   - Connect Telegram Trigger output to this node

7. **Add AI Agent Node ("AI Agent") for message summarization**  
   - Use Langchain agent node with prompt to summarize patient message into clear summaries with no fabricated info  
   - Configure with Google Gemini Chat Model node as LLM backend  
   - Connect Edit Fields output to AI Agent input

8. **Add AI Agent Node ("AI Agent4") for triage classification**  
   - Prompt to classify summaries into status (read/ignore) and intensity (low/moderate/high)  
   - Connect AI Agent output to AI Agent4 input  
   - Use Google Gemini Chat Model4 as backend

9. **Add Code Node ("Code1")**  
   - Parse AI Agent4 string output into JSON with keys status and intensity  
   - Connect AI Agent4 output to Code1 input

10. **Add If Node ("If")**  
    - Condition: status equals "read"  
    - True branch: connect to Switch node  
    - False branch: connect to feedback message node (Telegram)

11. **Add Switch Node ("Switch")**  
    - Routes based on intensity: high, moderate, low  
    - Connect If true output to Switch input

12. **Add AI Agent Nodes for intensity responses:**  
    - AI Agent1 (low) → Google Gemini Chat Model1  
    - AI Agent2 (moderate) → Google Gemini Chat Model2  
    - AI Agent3 (high) → Google Gemini Chat Model3  
    - Connect Switch outputs accordingly

13. **Add Telegram Send Nodes for each response:**  
    - Send a text message4 (low intensity) connected to AI Agent1  
    - Send a text message2 (moderate intensity) connected to AI Agent2  
    - Send a text message3 (high intensity) connected to AI Agent3  

14. **For moderate intensity:**  
    - Connect AI Agent2 output to AI Agent6 (appointment scheduling)  
    - AI Agent6 uses Google Gemini Chat Model6  
    - Connect AI Agent6 output to:  
      - Google Calendar node (Create an event)  
      - Telegram send node to notify patient

15. **For high intensity:**  
    - Connect AI Agent3 output to AI Agent5 (doctor email generation)  
    - AI Agent5 uses Google Gemini Chat Model5  
    - Connect AI Agent5 to:  
      - Google Sheets node to get doctor email  
      - Gmail node to send email  
      - HTTP Request node to initiate phone call via VAPI.ai API

16. **Set up all credentials:**  
    - Google Sheets OAuth2 for patient data  
    - Telegram API for messaging and triggers  
    - Google Palm API (Google Gemini) for AI agents  
    - Google Calendar OAuth2 for appointment creation  
    - Gmail OAuth2 for sending doctor emails  
    - VAPI.ai API key for automated calls

17. **Test each branch independently:**  
    - Ensure schedule trigger fires and messages send correctly  
    - Test Telegram incoming message processing and AI triage  
    - Test routing and responses for all intensity levels  
    - Test appointment creation and doctor notifications

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| AI-Powered Post-Surgery Patient Care & Communication: Daily automated patient outreach, AI triage, empathetic replies, appointment scheduling, and urgent notifications to doctors. Setup requires connecting Google Sheets, Telegram, Google Gemini (PaLM), Google Calendar, Gmail, and VAPI.ai APIs.               | Sticky Note at workflow start                                                                                                     |
| Setup checklist: Connect Google Sheets, Telegram, Google Gemini API key, Google Calendar, Gmail, and VAPI.ai API key. Configure daily schedule for outreach.                                                                                                                                               | Sticky Note at workflow start                                                                                                     |
| Workflow uses advanced AI prompt engineering to ensure no fabrication of patient history and maintains professional and empathetic tone.                                                                                                                                                                   | Multiple AI Agent nodes with detailed prompt templates                                                                              |
| Telegram chatId "5443133930" is hardcoded for outgoing messages; adjust to dynamic patient chat IDs for production.                                                                                                                                                                                        | Telegram Send nodes                                                                                                                |
| VAPI.ai API call requires valid Bearer token and correct phoneNumberId to initiate calls; replace placeholder values accordingly.                                                                                                                                                                          | HTTP Request node                                                                                                                  |
| Google Sheets document ID and sheet gid are specific to patient list; update to your own spreadsheet for deployment.                                                                                                                                                                                       | Google Sheets nodes                                                                                                                |
| Gmail node sends urgent emails to doctors; ensure doctor emails are stored in Google Sheets for retrieval.                                                                                                                                                                                                | AI Agent5 and Gmail node                                                                                                          |
| Appointment scheduling creates events 1-2 days ahead based on AI-generated data; ensure calendar linked to correct user account.                                                                                                                                                                          | AI Agent6 and Google Calendar node                                                                                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---