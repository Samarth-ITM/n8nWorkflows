Health Monitoring System with Grok-3 AI Analysis and Family/Doctor Email Alerts

https://n8nworkflows.xyz/workflows/health-monitoring-system-with-grok-3-ai-analysis-and-family-doctor-email-alerts-10525


# Health Monitoring System with Grok-3 AI Analysis and Family/Doctor Email Alerts

### 1. Workflow Overview

**Purpose:**  
This workflow automates patient health monitoring by receiving submitted health data, analyzing it with an AI-powered model (Grok-3 via OpenRouter), determining if alerts are necessary based on the analysis, and sending notification emails to the patient’s family and doctor when critical conditions are detected.

**Target Use Cases:**  
- Chronic disease management (e.g., diabetes glucose monitoring).  
- Elderly care with continuous vital sign tracking.  
- Remote patient monitoring systems requiring timely alerts.  
- Automated triage to reduce false alarms and improve clinical response times.

**Logical Blocks:**  
- **1.1 Input Reception & Data Extraction:** Receives health data via webhook and extracts relevant patient info.  
- **1.2 AI Health Data Analysis:** Uses Grok-3 AI model to analyze vitals, symptoms, and diet, generating structured health assessments.  
- **1.3 Alert Decision & Preparation:** Evaluates AI output flags to decide if alerts are required and prepares data for notifications.  
- **1.4 Notification Delivery:** Sends emails to family and doctor based on alert conditions.  
- **1.5 Result Consolidation & Response:** Merges results of alert sending, handles no-alert cases, and responds to the original webhook request.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Extraction

**Overview:**  
This block receives incoming health data via HTTP POST, extracts and structures the patient’s vital signs, symptoms, and contact info for subsequent analysis.

**Nodes Involved:**  
- Webhook - Submit Health Data  
- Extract Health Data

**Node Details:**

- **Webhook - Submit Health Data**  
  - Type: Webhook  
  - Role: Entry point API for receiving patient health data in JSON format via POST at path `/health-monitor`.  
  - Configuration: HTTP method POST, response mode immediate response node. No authentication configured by default (should be secured in production).  
  - Input: Incoming HTTP POST requests with JSON body containing patient info and vitals.  
  - Output: Passes raw JSON data downstream.  
  - Potential Failures: Missing fields, malformed JSON, unauthorized access if no auth configured.

- **Extract Health Data**  
  - Type: Set  
  - Role: Extracts and maps fields from nested JSON body into flattened workflow variables for clarity and downstream use.  
  - Configuration: Maps patientName, age, bloodPressure, heartRate, weight, height, dietLog, symptoms, familyEmail, doctorEmail from `$json.body`.  
  - Expressions: Uses `={{ $json.body.<field> }}` to assign values.  
  - Input: Raw webhook JSON payload.  
  - Output: Structured item with direct patient and contact fields.  
  - Edge Cases: Missing or null fields could cause incomplete data; no validation on data formats here.

---

#### 1.2 AI Health Data Analysis

**Overview:**  
Analyzes extracted health data using the Grok-3 AI model accessed via OpenRouter. Produces a structured health assessment including risk levels and alert flags.

**Nodes Involved:**  
- AI Health Analysis Agent  
- OpenRouter Chat Model  
- Structured Output Parser

**Node Details:**

- **AI Health Analysis Agent**  
  - Type: LangChain Agent  
  - Role: Sends patient data prompt to AI model, requesting a JSON-formatted health assessment.  
  - Configuration:  
    - Prompt includes patient vitals, symptoms, diet log, with instructions to output a detailed JSON including healthStatus, riskLevel, recommendations, and alert flags.  
    - System message defines AI as a medical assistant specializing in preventive care.  
    - Uses structured output parser for JSON.  
  - Input: Patient data from "Extract Health Data".  
  - Output: AI-generated structured JSON with health analysis and alert decisions.  
  - Edge Cases: AI API failures, rate limits, malformed AI output, or incomplete JSON responses.

- **OpenRouter Chat Model**  
  - Type: Language Model (OpenRouter)  
  - Role: Provides the AI model endpoint "x-ai/grok-3" for the agent to use.  
  - Configuration: Requires OpenRouter API credentials.  
  - Input: Prompt from AI Health Analysis Agent.  
  - Output: AI text response.  
  - Possible Failures: API authentication errors, network issues, model unavailability.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI text output into usable JSON format for workflow logic.  
  - Input: AI model raw output.  
  - Output: Parsed JSON fields accessible in workflow.  
  - Failures: Parsing errors if AI output deviates from expected JSON format.

---

#### 1.3 Alert Decision & Preparation

**Overview:**  
Evaluates AI flags to determine if alerts should be sent to family and/or doctor. Prepares the alert data payload for email notifications.

**Nodes Involved:**  
- Check If Alert Needed  
- Prepare Alert Data  
- Check If Doctor Alert Needed  
- No Alert Required

**Node Details:**

- **Check If Alert Needed**  
  - Type: If (Boolean condition)  
  - Role: Checks if AI output flag `alertFamily` is true to proceed with alerts.  
  - Configuration: Boolean condition on `{{$json.output.alertFamily}} === true` (strict).  
  - Input: AI analysis output.  
  - Output:  
    - True: Continue to alert preparation.  
    - False: Route to "No Alert Required".  
  - Edge Cases: Missing alert flags or incorrect boolean parsing.

- **Prepare Alert Data**  
  - Type: Set  
  - Role: Gathers patient and AI analysis details into a clean alert data structure for emails.  
  - Configuration: Assigns familyEmail, doctorEmail, patientName, healthStatus, riskLevel, healthSummary, recommendations, vitalsConcerns, dietAnalysis, and alertDoctor flag from previous nodes.  
  - Input: From "Check If Alert Needed" true path.  
  - Output: Structured alert data for email nodes.

- **Check If Doctor Alert Needed**  
  - Type: If  
  - Role: Checks if AI output flag `alertDoctor` is true to decide sending a medical alert email.  
  - Configuration: Boolean condition on `{{$json.alertDoctor}} === true`.  
  - Input: After "Send Email to Family".  
  - Output:  
    - True: Proceed to send doctor email.  
    - False: Skip to merging results directly.  
  - Edge Cases: Missing doctor email or flag.

- **No Alert Required**  
  - Type: Set  
  - Role: Creates a no-alert status record when AI indicates no alert is needed.  
  - Configuration: Sets status to `no_alert_needed`, includes message "Health status is within normal parameters. No alerts required." and preserves healthStatus.  
  - Input: From "Check If Alert Needed" false path.  
  - Output: Status message for final response.

---

#### 1.4 Notification Delivery

**Overview:**  
Sends email notifications to family and doctor based on alert conditions.

**Nodes Involved:**  
- Send Email to Family  
- Send Email to Doctor

**Node Details:**

- **Send Email to Family**  
  - Type: Email Send  
  - Role: Sends a health alert email to the patient’s family email address.  
  - Configuration:  
    - Subject: "⚠️ Health Alert for {patientName}" (dynamic).  
    - To: familyEmail from alert data.  
    - From: fixed sender "health-monitor@yourdomain.com".  
  - Input: From "Prepare Alert Data".  
  - Output: Email send success status.  
  - Failures: SMTP authentication issues, invalid email addresses, network failures.

- **Send Email to Doctor**  
  - Type: Email Send  
  - Role: Sends a medical alert email to the patient’s doctor if doctor alert is flagged.  
  - Configuration:  
    - Subject: "Medical Alert: {patientName} - {RISKLEVEL} Risk" (riskLevel uppercased).  
    - To: doctorEmail from alert data.  
    - From: fixed sender "health-monitor@yourdomain.com".  
  - Input: From "Check If Doctor Alert Needed" true path.  
  - Output: Email send success status.  
  - Failures: Same as family email node.

---

#### 1.5 Result Consolidation & Response

**Overview:**  
Combines alert sending results or no-alert status and sends a JSON response to the original webhook caller.

**Nodes Involved:**  
- Merge Alert Results  
- Combine Results  
- Respond to Webhook

**Node Details:**

- **Merge Alert Results**  
  - Type: Set  
  - Role: Consolidates results from email nodes, sets status and flags indicating if family and doctor alerts were sent successfully.  
  - Configuration:  
    - Sets `status = alerts_sent`  
    - `familyAlerted` and `doctorAlerted` booleans based on email success flags.  
    - Message confirming alerts sent.  
  - Input: From email nodes outputs.  
  - Output: Status object for final combination.

- **Combine Results**  
  - Type: Merge (combine mode)  
  - Role: Combines outputs either from "No Alert Required" or "Merge Alert Results" to unify the workflow output.  
  - Configuration: Combine all inputs into one output item.  
  - Input: Outputs from alert or no alert paths.  
  - Output: Single combined JSON object with status and message.

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends final JSON response to the original webhook caller.  
  - Configuration: Responds with JSON including:  
    - success: true  
    - analysisComplete: true  
    - healthStatus (from alerts or AI agent)  
    - alertsSent: boolean depending on status  
    - message: status message from previous nodes  
  - Input: From "Combine Results".  
  - Output: HTTP response to client.  
  - Edge Cases: Response failures if workflow crashes before this node.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                          | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                                             |
|----------------------------|---------------------------------|----------------------------------------|--------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Webhook - Submit Health Data | Webhook                         | Receives patient health data via POST | (start)                        | Extract Health Data               | Automates patient health monitoring by analyzing submitted health data via AI, determining alert necessity, and notifying family and doctors when critical conditions detected.         |
| Extract Health Data         | Set                             | Extracts and structures input data     | Webhook - Submit Health Data   | AI Health Analysis Agent          |                                                                                                                                         |
| OpenRouter Chat Model       | Language Model (OpenRouter)      | Provides Grok-3 AI model endpoint      | AI Health Analysis Agent (lm)  | AI Health Analysis Agent (lmOut) |                                                                                                                                         |
| AI Health Analysis Agent    | LangChain Agent                 | Runs AI health assessment               | Extract Health Data            | Check If Alert Needed             |                                                                                                                                         |
| Structured Output Parser    | LangChain Output Parser          | Parses AI output to JSON                | AI Health Analysis Agent       | AI Health Analysis Agent          |                                                                                                                                         |
| Check If Alert Needed       | If                              | Determines if family alert required     | AI Health Analysis Agent       | Prepare Alert Data, No Alert Required |                                                                                                                                         |
| Prepare Alert Data          | Set                             | Prepares alert data for emails          | Check If Alert Needed (true)   | Send Email to Family              |                                                                                                                                         |
| Send Email to Family        | Email Send                      | Sends alert email to family             | Prepare Alert Data             | Check If Doctor Alert Needed      |                                                                                                                                         |
| Check If Doctor Alert Needed| If                              | Determines if doctor alert required     | Send Email to Family           | Send Email to Doctor, Merge Alert Results |                                                                                                                                         |
| Send Email to Doctor        | Email Send                      | Sends alert email to doctor             | Check If Doctor Alert Needed   | Merge Alert Results               |                                                                                                                                         |
| Merge Alert Results         | Set                             | Combines email send success statuses   | Send Email to Doctor           | Combine Results                  |                                                                                                                                         |
| No Alert Required           | Set                             | Sets no alert status when not needed    | Check If Alert Needed (false)  | Combine Results                  |                                                                                                                                         |
| Combine Results            | Merge                           | Combines alert/no-alert results         | Merge Alert Results, No Alert Required | Respond to Webhook               |                                                                                                                                         |
| Respond to Webhook          | Respond to Webhook              | Final HTTP response to webhook caller  | Combine Results               | (end)                           |                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: "Webhook - Submit Health Data"  
   - HTTP Method: POST  
   - Path: "health-monitor"  
   - Response Mode: "Response Node" (to send response after workflow completes)  
   - Secure with authentication (recommended for production).  

2. **Create Set Node to Extract Data:**  
   - Name: "Extract Health Data"  
   - Map JSON fields from webhook body:  
     - patientName (string): `={{ $json.body.patientName }}`  
     - age (number): `={{ $json.body.age }}`  
     - bloodPressure (string): `={{ $json.body.bloodPressure }}`  
     - heartRate (number): `={{ $json.body.heartRate }}`  
     - weight (number): `={{ $json.body.weight }}`  
     - height (number): `={{ $json.body.height }}`  
     - dietLog (string): `={{ $json.body.dietLog }}`  
     - symptoms (string): `={{ $json.body.symptoms }}`  
     - familyEmail (string): `={{ $json.body.familyEmail }}`  
     - doctorEmail (string): `={{ $json.body.doctorEmail }}`  

3. **Create OpenRouter Chat Model Node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
   - Name: "OpenRouter Chat Model"  
   - Model: "x-ai/grok-3"  
   - Credentials: Configure OpenRouter API key credentials.  

4. **Create AI Health Analysis Agent Node:**  
   - Type: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)  
   - Name: "AI Health Analysis Agent"  
   - Connect OpenRouter Chat Model as language model in node settings.  
   - Prompt Text: Include detailed patient vitals and request JSON output with fields: healthStatus, riskLevel, healthSummary, dietAnalysis, vitalsConcerns, recommendations, requiresImmediateAttention, alertFamily, alertDoctor.  
   - System Message: Clarify AI role as medical assistant specializing in health monitoring and preventive care.  
   - Output parser: Use structured output parser for JSON.  

5. **Create Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)  
   - Name: "Structured Output Parser"  
   - Link this parser node to the AI Health Analysis Agent’s output parser setting.  

6. **Create If Node "Check If Alert Needed":**  
   - Type: If  
   - Name: "Check If Alert Needed"  
   - Condition: Boolean check if `{{$json.output.alertFamily}} === true`  
   - True path: proceed to alert preparation.  
   - False path: proceed to no alert.  

7. **Create Set Node "Prepare Alert Data":**  
   - Name: "Prepare Alert Data"  
   - Assign variables from extracted data and AI output: familyEmail, doctorEmail, patientName, healthStatus, riskLevel, healthSummary, recommendations, vitalsConcerns, dietAnalysis, alertDoctor.  

8. **Create Email Send Node "Send Email to Family":**  
   - SMTP or Gmail credentials configured.  
   - To: `={{ $json.familyEmail }}`  
   - From: "health-monitor@yourdomain.com"  
   - Subject: "⚠️ Health Alert for {{ $json.patientName }}"  

9. **Create If Node "Check If Doctor Alert Needed":**  
   - Condition: Boolean check if `{{$json.alertDoctor}} === true`  
   - True path: send doctor email.  
   - False path: skip to merge results.  

10. **Create Email Send Node "Send Email to Doctor":**  
    - To: `={{ $json.doctorEmail }}`  
    - From: "health-monitor@yourdomain.com"  
    - Subject: "Medical Alert: {{ $json.patientName }} - {{ $json.riskLevel.toUpperCase() }} Risk"  

11. **Create Set Node "Merge Alert Results":**  
    - Set status to "alerts_sent"  
    - Set `familyAlerted` and `doctorAlerted` booleans based on email send success flags.  
    - Add message "Health alerts have been sent successfully."  

12. **Create Set Node "No Alert Required":**  
    - Status: "no_alert_needed"  
    - Message: "Health status is within normal parameters. No alerts required."  
    - healthStatus from AI output.  

13. **Create Merge Node "Combine Results":**  
    - Mode: Combine (combine all inputs)  
    - Combine outputs of "Merge Alert Results" and "No Alert Required".  

14. **Create Respond to Webhook Node:**  
    - Respond with JSON status including: success, analysisComplete, healthStatus, alertsSent (true if alerts sent), and message.  

15. **Connect Nodes in order:**  
    - Webhook → Extract Health Data → AI Health Analysis Agent  
    - AI Health Analysis Agent → Check If Alert Needed  
    - Check If Alert Needed true → Prepare Alert Data → Send Email to Family → Check If Doctor Alert Needed  
    - Check If Doctor Alert Needed true → Send Email to Doctor → Merge Alert Results  
    - Check If Doctor Alert Needed false → Merge Alert Results  
    - Check If Alert Needed false → No Alert Required  
    - Merge Alert Results & No Alert Required → Combine Results → Respond to Webhook  

16. **Configure Credentials:**  
    - OpenRouter API credentials for AI nodes.  
    - SMTP or Gmail credentials for Email Send nodes.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Automates patient health monitoring by analyzing submitted health data via AI, determining alert necessity, and notifying family and doctors if critical. | Sticky Note near Webhook node - introduction and workflow summary.                                               |
| Use Cases: Chronic disease management (e.g., diabetes glucose monitoring), elderly care, etc.                                                             | Sticky Note near alert and email nodes describing practical applications.                                        |
| Customization tips: Adjust alert thresholds by demographics, add vitals, customize AI prompts, integrate SMS via Twilio, connect EHR, add escalation logic.| Sticky Note near alert nodes offering workflow extension ideas.                                                  |
| Benefits include rapid emergency detection, intelligent filtering to reduce false alarms, family peace of mind, and clinical efficiency improvement.      | Sticky Note near alert nodes emphasizing workflow advantages.                                                    |
| Setup instructions: Secure webhook, configure OpenRouter API key and AI model, set up SMTP for email, define alert criteria (e.g., temp >38.5°C, HR limits). | Sticky Note near AI and webhook nodes with setup guidelines and prerequisites.                                   |

---

This documentation fully describes the workflow structure, logic, node configurations, and setup instructions enabling reproduction, modification, and troubleshooting by advanced users or AI agents.