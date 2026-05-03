Medical Triage & Appointment Automation with GPT-4 and JotForm

https://n8nworkflows.xyz/workflows/medical-triage---appointment-automation-with-gpt-4-and-jotform-9446


# Medical Triage & Appointment Automation with GPT-4 and JotForm

---

## 1. Workflow Overview

This workflow automates the medical patient intake and triage process using AI-driven analysis powered by GPT-4 and integrates with JotForm for data collection. It targets healthcare providers aiming to streamline appointment prioritization by classifying urgency levels (emergency, urgent, routine, non-urgent), generating detailed pre-appointment briefs, and automating communication and scheduling notifications.

### Logical Blocks:

- **1.1 Patient Intake Collection**: Captures structured patient data from a HIPAA-compliant JotForm submission.
- **1.2 Patient Data Processing**: Calculates patient age and categorizes them based on age groups.
- **1.3 AI Medical Triage & Analysis**: Uses GPT-4 to analyze patient data and generate a comprehensive triage report including urgency assessment, possible conditions, and recommendations.
- **1.4 Emergency Handling**: Detects emergency cases to trigger immediate alerts via Slack and email to emergency teams and the on-call doctor.
- **1.5 Urgent Scheduling**: Notifies front desk staff and sends urgent appointment confirmation emails to patients requiring fast scheduling.
- **1.6 Routine Scheduling**: Notifies scheduling staff and sends routine appointment confirmation emails for non-urgent cases.
- **1.7 Patient Data Logging**: Records final patient data, triage results, and appointment details into a Google Sheet for tracking, analytics, and compliance.

---

## 2. Block-by-Block Analysis

### 2.1 Patient Intake Collection

**Overview:**  
Captures comprehensive patient information submitted through a JotForm medical intake form and structures it into relevant fields for downstream processing.

**Nodes Involved:**  
- JotForm Trigger1  
- Extract Patient Data  
- Sticky Note - Intake

**Node Details:**

- **JotForm Trigger1**  
  - Type: Trigger node (JotForm Trigger)  
  - Role: Listens for new submissions on a specified JotForm form ID.  
  - Configuration: Connected to a specific form (ID: 252815075257460) with authenticated JotForm API credentials.  
  - Inputs: None (trigger event).  
  - Outputs: Raw JSON data from form submission.  
  - Edge Cases: API connectivity issues, form deactivation, or permission errors.

- **Extract Patient Data**  
  - Type: Set node  
  - Role: Maps raw submission data into named variables for patient demographics, symptoms, medical history, insurance, and appointment preferences.  
  - Configuration: Uses expressions to extract and default patient fields (e.g., patient_name concatenates first and last names; defaults provided for optional fields like pain_level).  
  - Inputs: JotForm Trigger1 output.  
  - Outputs: Structured JSON with patient-specific keys for use in subsequent nodes.  
  - Edge Cases: Missing or malformed data fields could result in default or blank values.

- **Sticky Note - Intake**  
  - Type: Sticky Note  
  - Role: Documentation block providing context about the intake data capture process and a link to create JotForm.  
  - No execution role.

---

### 2.2 Patient Data Processing

**Overview:**  
Calculates patient age from date of birth, classifies the patient into age categories (infant, child, adolescent, adult, senior) to inform AI analysis.

**Nodes Involved:**  
- Calculate Patient Info

**Node Details:**

- **Calculate Patient Info**  
  - Type: Code node (JavaScript)  
  - Role: Processes input JSON to compute age and assign patient category.  
  - Configuration: Uses current date and date_of_birth field; categorizes age groups with if-else logic.  
  - Inputs: Extract Patient Data output.  
  - Outputs: Augmented JSON including `patient_age` and `patient_category`.  
  - Edge Cases: Invalid or missing date_of_birth can cause parsing errors or NaN ages; no explicit validation implemented.

---

### 2.3 AI Medical Triage & Analysis

**Overview:**  
Leverages GPT-4 through the LangChain agent node to interpret patient data, perform urgency classification, symptom analysis, differential diagnosis, and generate provider and patient guidance.

**Nodes Involved:**  
- AI Medical Triage & Analysis  
- OpenAI Chat Model  
- Structured Output Parser  
- Sticky Note - Triage

**Node Details:**

- **AI Medical Triage & Analysis**  
  - Type: LangChain Agent Node  
  - Role: Sends a detailed prompt with patient data to GPT-4 for medical triage analysis.  
  - Configuration: Provides a multi-section prompt template containing patient demographics, symptoms, medical background, and detailed instructions for response structure including urgency levels, symptom summaries, possible conditions, alerts, provider recommendations, and appointment details.  
  - Inputs: Calculate Patient Info output.  
  - Outputs: Raw AI response JSON.  
  - Edge Cases: API rate limits, model availability, prompt interpretation errors, latency; requires GPT-4 capable environment.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Node  
  - Role: Executes GPT-4 model inference with temperature set to 0.3 for controlled creativity.  
  - Configuration: Uses "gpt-4o" model with OpenAI API credentials.  
  - Inputs: AI Medical Triage & Analysis.  
  - Outputs: AI response passed to parser.  
  - Edge Cases: API authentication failure, quota exhaustion.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI response into a strict JSON schema with required fields such as urgency_level, symptom_summary, critical_alerts, appointment_duration, and detailed markdown analysis.  
  - Inputs: OpenAI Chat Model output.  
  - Outputs: Parsed structured data for conditional logic and notifications.  
  - Edge Cases: Parsing failure if AI response deviates from schema; schema validation errors.

- **Sticky Note - Triage**  
  - Type: Sticky Note  
  - Role: Describes the AI triage function and its output as a comprehensive pre-appointment brief.

---

### 2.4 Emergency Handling

**Overview:**  
If the AI triage classifies the case as an emergency, triggers immediate alerts to emergency teams via Slack, sends urgent patient instructions email, and notifies the on-call doctor.

**Nodes Involved:**  
- Is Emergency? (If node)  
- Alert Emergency Team (Slack)  
- Send Emergency Instructions  
- Alert On-Call Doctor  
- Sticky Note - Emergency

**Node Details:**

- **Is Emergency?**  
  - Type: If node  
  - Role: Checks if `urgency_level` equals "emergency".  
  - Inputs: AI Medical Triage & Analysis output.  
  - Outputs: True branch triggers emergency notifications.  
  - Edge Cases: Missing urgency_level field.

- **Alert Emergency Team (Slack)**  
  - Type: HTTP Request node  
  - Role: Posts a formatted emergency alert message to a Slack channel (#medical-emergencies) with patient details, priority score, symptoms, red flags, and instructions.  
  - Configuration: Uses Slack API token via HTTP header authentication; message includes rich markdown and Slack blocks for formatting.  
  - Inputs: Is Emergency? True branch.  
  - Outputs: Triggers next node to send email instructions.  
  - Edge Cases: Slack API rate limits, authentication errors.

- **Send Emergency Instructions**  
  - Type: Gmail node  
  - Role: Sends an urgent HTML email to the patient explaining the emergency status and instructing to call 911 or go to ER immediately.  
  - Configuration: Uses Gmail OAuth2 credentials; email includes detailed styled content with warnings, contact info, and instructions.  
  - Inputs: Alert Emergency Team (Slack) output.  
  - Outputs: Triggers Alert On-Call Doctor email node.  
  - Edge Cases: Email delivery failure, invalid patient email.

- **Alert On-Call Doctor**  
  - Type: Gmail node  
  - Role: Sends a detailed HTML email alert to the on-call doctor with patient info, AI analysis, red flags, and priority details for immediate review.  
  - Configuration: Uses Gmail OAuth2; email formatted with tables and styled sections for clarity.  
  - Inputs: Send Emergency Instructions output.  
  - Outputs: Leads to logging.  
  - Edge Cases: Email failure, incorrect recipient address.

- **Sticky Note - Emergency**  
  - Type: Sticky Note  
  - Role: Documents emergency protocol emphasizing patient safety and rapid response (<15 minutes callback).

---

### 2.5 Urgent Scheduling

**Overview:**  
For cases classified as urgent (but not emergency), notifies front desk to schedule an appointment within 24-48 hours and sends patient confirmation with instructions.

**Nodes Involved:**  
- Is Urgent? (If node)  
- Notify Front Desk (Urgent)  
- Send Patient Confirmation (Urgent)  
- Sticky Note - Urgent

**Node Details:**

- **Is Urgent?**  
  - Type: If node  
  - Role: Checks if `urgency_level` equals "urgent".  
  - Inputs: AI Medical Triage & Analysis output.  
  - Outputs: True branch initiates urgent scheduling notifications.  
  - Edge Cases: Missing urgency_level.

- **Notify Front Desk (Urgent)**  
  - Type: Gmail node  
  - Role: Sends an urgent scheduling request email to front-desk@clinic.com with patient data, priority score, symptoms, and recommended provider.  
  - Configuration: Uses Gmail OAuth2 credentials; email contains styled HTML with instructions to contact patient and schedule within 24-48 hours.  
  - Inputs: Is Urgent? True branch.  
  - Outputs: Triggers sending patient confirmation email.  
  - Edge Cases: Email delivery failure.

- **Send Patient Confirmation (Urgent)**  
  - Type: Gmail node  
  - Role: Sends a confirmation email to the patient notifying them of urgent status and that scheduling contact will occur soon.  
  - Configuration: Uses Gmail OAuth2; email includes instructions on what to bring and pre-appointment preparations.  
  - Inputs: Notify Front Desk (Urgent) output.  
  - Outputs: Proceeds to patient data logging.  
  - Edge Cases: Invalid patient email.

- **Sticky Note - Urgent**  
  - Type: Sticky Note  
  - Role: Notes the goal of fast scheduling within 24-48 hours without alarming the patient unnecessarily.

---

### 2.6 Routine Scheduling

**Overview:**  
Handles routine or non-urgent cases by notifying schedulers to book appointments within 1-2 weeks and sending patient confirmation emails accordingly.

**Nodes Involved:**  
- Notify Scheduler (Routine)  
- Send Patient Confirmation (Routine)  
- Sticky Note - Routine

**Node Details:**

- **Notify Scheduler (Routine)**  
  - Type: Gmail node  
  - Role: Sends appointment scheduling request to scheduler@clinic.com with patient details, recommended provider, and instructions to schedule within 1-2 weeks.  
  - Configuration: Uses Gmail OAuth2; email styled with patient data and appointment instructions.  
  - Inputs: Is Urgent? False branch (connected from Is Urgent? node).  
  - Outputs: Triggers sending patient confirmation email.  
  - Edge Cases: Email failures.

- **Send Patient Confirmation (Routine)**  
  - Type: Gmail node  
  - Role: Sends a polite confirmation email to the patient acknowledging form receipt and promising scheduling contact within 1-2 business days.  
  - Configuration: Uses Gmail OAuth2; includes appointment preferences and preparation instructions.  
  - Inputs: Notify Scheduler (Routine) output.  
  - Outputs: Leads to patient data logging.  
  - Edge Cases: Email delivery issues.

- **Sticky Note - Routine**  
  - Type: Sticky Note  
  - Role: Describes routine scheduling goal for smooth experience and thorough preparation.

---

### 2.7 Patient Data Logging

**Overview:**  
Appends all patient intake data, AI triage results, and appointment decision metadata into a Google Sheet for record-keeping, analytics, and compliance.

**Nodes Involved:**  
- Log to Patient Database  
- Sticky Note - Database

**Node Details:**

- **Log to Patient Database**  
  - Type: Google Sheets node  
  - Role: Appends a new row with patient demographics, symptoms, AI output fields (urgency, priority score), appointment duration, and provider recommendations.  
  - Configuration: Uses Google Sheets API with OAuth2 credentials; targets a specific sheet ID and sheet name (gid=0).  
  - Inputs: Outputs of Send Patient Confirmation (Urgent), Send Patient Confirmation (Routine), and Alert On-Call Doctor nodes.  
  - Outputs: None (terminal node).  
  - Edge Cases: API quota limits, sheet access permissions, data format mismatches.

- **Sticky Note - Database**  
  - Type: Sticky Note  
  - Role: Explains the purpose of the patient data logging for compliance and visibility.

---

## 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                   | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                   |
|-----------------------------|----------------------------------|--------------------------------------------------|---------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------|
| JotForm Trigger1             | JotForm Trigger                  | Trigger on new patient intake submission         | None                            | Extract Patient Data             | 📋 Patient Intake Collection: Captures comprehensive patient info via JotForm with HIPAA compliance.           |
| Extract Patient Data         | Set                             | Extracts and structures patient form data        | JotForm Trigger1                | Calculate Patient Info            | Same as above                                                                                                |
| Calculate Patient Info       | Code                            | Calculates patient age and categorizes age group | Extract Patient Data            | AI Medical Triage & Analysis      |                                                                                                               |
| AI Medical Triage & Analysis | LangChain Agent                 | Sends patient data to GPT-4 for triage analysis  | Calculate Patient Info           | Is Emergency?, Is Urgent?         | 🤖 AI Medical Triage & Analysis: AI generates provider briefs and patient guidance.                           |
| OpenAI Chat Model            | LangChain OpenAI Chat Model     | Executes GPT-4 inference                          | AI Medical Triage & Analysis     | Structured Output Parser          |                                                                                                               |
| Structured Output Parser     | LangChain Output Parser         | Parses AI response into structured JSON           | OpenAI Chat Model               | AI Medical Triage & Analysis      |                                                                                                               |
| Is Emergency?                | If                             | Checks if urgency level is emergency              | AI Medical Triage & Analysis     | Alert Emergency Team (Slack)      | 🚨 Emergency Response Protocol: Patient safety prioritized, <15 min doctor callback.                          |
| Alert Emergency Team (Slack) | HTTP Request                   | Sends emergency alert message to Slack channel   | Is Emergency?                   | Send Emergency Instructions       | Same as above                                                                                                |
| Send Emergency Instructions  | Gmail                          | Sends emergency email instructions to patient    | Alert Emergency Team (Slack)    | Alert On-Call Doctor              | Same as above                                                                                                |
| Alert On-Call Doctor         | Gmail                          | Alerts on-call doctor with emergency details      | Send Emergency Instructions     | Log to Patient Database          | Same as above                                                                                                |
| Is Urgent?                   | If                             | Checks if urgency level is urgent                 | AI Medical Triage & Analysis     | Notify Front Desk (Urgent), Notify Scheduler (Routine) | ⚡ Urgent Scheduling Path: Fast scheduling within 24-48 hours without alarming patient unnecessarily.          |
| Notify Front Desk (Urgent)   | Gmail                          | Notifies front desk for urgent scheduling         | Is Urgent? True branch          | Send Patient Confirmation (Urgent) | Same as above                                                                                                |
| Send Patient Confirmation (Urgent) | Gmail                  | Sends urgent appointment confirmation to patient | Notify Front Desk (Urgent)      | Log to Patient Database          | Same as above                                                                                                |
| Notify Scheduler (Routine)   | Gmail                          | Notifies scheduler for routine appointment        | Is Urgent? False branch         | Send Patient Confirmation (Routine) | 📅 Routine Scheduling Path: Smooth scheduling within 1-2 weeks with complete info.                             |
| Send Patient Confirmation (Routine) | Gmail                  | Sends routine appointment confirmation to patient| Notify Scheduler (Routine)      | Log to Patient Database          | Same as above                                                                                                |
| Log to Patient Database      | Google Sheets                  | Logs patient and triage data for record-keeping  | Send Patient Confirmation (Urgent), Send Patient Confirmation (Routine), Alert On-Call Doctor | None                           | 📊 Patient Data Management: Logging for analytics, compliance, and patient journey visibility.                |
| Sticky Note - Intake         | Sticky Note                    | Documentation of intake data capture              | None                           | None                            | See above                                                                                                    |
| Sticky Note - Triage         | Sticky Note                    | Documentation of AI triage process                 | None                           | None                            | See above                                                                                                    |
| Sticky Note - Emergency      | Sticky Note                    | Documentation of emergency handling protocol       | None                           | None                            | See above                                                                                                    |
| Sticky Note - Urgent         | Sticky Note                    | Documentation of urgent scheduling path            | None                           | None                            | See above                                                                                                    |
| Sticky Note - Routine        | Sticky Note                    | Documentation of routine scheduling path           | None                           | None                            | See above                                                                                                    |
| Sticky Note - Database       | Sticky Note                    | Documentation of patient data logging              | None                           | None                            | See above                                                                                                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Set form ID to your medical intake form (e.g., 252815075257460).  
   - This node listens to new form submissions.

2. **Add Set Node: Extract Patient Data**  
   - Type: Set  
   - Map incoming data fields to variables: patient_name (concatenate first and last name), patient_email, patient_phone, date_of_birth, reason_for_visit, symptoms, symptom_duration, pain_level (default "N/A"), current_medications (default "None"), allergies (default "None"), medical_history (default "None reported"), insurance_provider (default "Self-pay"), preferred_date, preferred_time.  
   - Create unique intake_id as `MED-<submissionID>-<timestamp>`.  
   - Add submission_date as current ISO string.

3. **Add Code Node: Calculate Patient Info**  
   - Type: Code (JavaScript)  
   - Input: Output of Extract Patient Data.  
   - Logic: Parse date_of_birth, calculate age, determine patient_category with rules: infant (<2), child (<12), adolescent (<18), adult (default), senior (>=65).  
   - Output augmented JSON with patient_age and patient_category.

4. **Set up LangChain Agent Node: AI Medical Triage & Analysis**  
   - Use LangChain agent with OpenAI GPT-4.  
   - Prompt template includes detailed patient info and instructions for a structured medical triage analysis (urgency classification, symptom analysis, differential diagnosis, alerts, recommendations, appointment details).  
   - Link to GPT-4 model via LangChain OpenAI Chat node, set temperature to 0.3 for controlled output.  
   - Use Structured Output Parser with a manual JSON schema enforcing required fields like urgency_level, priority_score, symptom_summary, red_flag_symptoms, appointment_duration, etc.

5. **Add If Node: Is Emergency?**  
   - Condition: `$json.output.urgency_level == 'emergency'`.  
   - True branch routes to emergency alert nodes.

6. **Add HTTP Request Node: Alert Emergency Team (Slack)**  
   - POST to Slack API endpoint `https://slack.com/api/chat.postMessage`.  
   - Authenticate using Slack token in HTTP header.  
   - JSON body includes patient details, priority score, symptoms, red flags, and emergency instructions formatted with Slack blocks.  
   - Connect output to next node.

7. **Add Gmail Node: Send Emergency Instructions**  
   - Email sent to patient_email with urgent instructions to call 911 or visit ER immediately.  
   - Styled HTML content includes warnings, contact info, red flags, and intake ID.  
   - Use Gmail OAuth2 credentials.

8. **Add Gmail Node: Alert On-Call Doctor**  
   - Sends detailed emergency alert to on-call-doctor@clinic.com with patient info, AI analysis, priority score, and recommended actions.  
   - Styled HTML email with tables and highlights.

9. **Add If Node: Is Urgent?**  
   - Condition: `$json.output.urgency_level == 'urgent'`.  
   - True branch routes to urgent scheduling nodes.  
   - False branch proceeds to routine scheduling.

10. **Add Gmail Node: Notify Front Desk (Urgent)**  
    - Sends urgent scheduling request email to front-desk@clinic.com with patient details, priority score, and instructions to schedule within 24-48 hours.  
    - Uses Gmail OAuth2.

11. **Add Gmail Node: Send Patient Confirmation (Urgent)**  
    - Sends urgent status confirmation email to patient with instructions on what to expect and bring.  
    - Uses Gmail OAuth2.

12. **Add Gmail Node: Notify Scheduler (Routine)**  
    - Sends routine appointment scheduling request to scheduler@clinic.com with patient details and instructions to schedule within 1-2 weeks.  
    - Uses Gmail OAuth2.

13. **Add Gmail Node: Send Patient Confirmation (Routine)**  
    - Sends routine confirmation email to patient acknowledging receipt and scheduling timeline.  
    - Uses Gmail OAuth2.

14. **Add Google Sheets Node: Log to Patient Database**  
    - Append patient and triage data including urgency, priority score, symptoms, allergies, appointment details to a Google Sheet.  
    - Configure with Google Sheets OAuth2 credentials.  
    - Specify target document ID and sheet name (gid=0).

15. **Connect nodes accordingly:**  
    - JotForm Trigger1 → Extract Patient Data → Calculate Patient Info → AI Medical Triage & Analysis →  
      → Is Emergency? (True → Alert Emergency Team (Slack) → Send Emergency Instructions → Alert On-Call Doctor → Log to Patient Database)  
      → Is Emergency? (False → Is Urgent?)  
      → Is Urgent? (True → Notify Front Desk (Urgent) → Send Patient Confirmation (Urgent) → Log to Patient Database)  
      → Is Urgent? (False → Notify Scheduler (Routine) → Send Patient Confirmation (Routine) → Log to Patient Database)

16. **Add Sticky Notes** at appropriate places to document blocks for readability.

---

## 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Patient intake form created on JotForm with HIPAA compliance for secure data capture.                            | [Create your form for free on JotForm](https://www.jotform.com/?partner=mediajade)              |
| AI Medical Triage prompt crafted to ensure comprehensive, structured, and safe medical analysis and recommendations. | Internal prompt text embedded in AI Medical Triage & Analysis node.                             |
| Emergency protocol prioritizes patient safety with rapid alerts via Slack and email to emergency teams.          | Emergency block Sticky Note describes <15 minute callback goal.                                |
| Urgent scheduling balances speed with patient reassurance to avoid unnecessary alarm.                            | Urgent scheduling Sticky Note highlights 24-48 hour scheduling goal.                           |
| Routine scheduling ensures smooth appointment booking with detailed provider prep information.                   | Routine scheduling Sticky Note documents 1-2 week scheduling goal.                             |
| Patient data logging enables analytics, compliance, and continuity of care tracking.                             | Database Sticky Note explains comprehensive patient journey visibility.                        |

---

**Disclaimer:** The provided text stems exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---