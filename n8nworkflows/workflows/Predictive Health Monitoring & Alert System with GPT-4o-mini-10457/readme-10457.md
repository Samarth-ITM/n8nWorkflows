Predictive Health Monitoring & Alert System with GPT-4o-mini

https://n8nworkflows.xyz/workflows/predictive-health-monitoring---alert-system-with-gpt-4o-mini-10457


# Predictive Health Monitoring & Alert System with GPT-4o-mini

### 1. Workflow Overview

This workflow implements a **Predictive Health Monitoring & Alert System** powered by advanced AI (GPT-4o-mini) to intake real-time wearable and patient health data, analyze it for abnormalities and trends, generate personalized health reports, and trigger alerts and follow-up actions as needed.

The system targets use cases such as continuous health monitoring for chronic disease management, elderly care, corporate wellness programs, and post-surgery recovery tracking. It automates early detection of health risks, personalized recommendations, and communication with patients and healthcare teams.

**Logical Blocks:**

- **1.1 Input Reception & Data Normalization:** Receive incoming health data via webhook, standardize it, and store raw and normalized data.
- **1.2 Historical Data Retrieval:** Query recent health records for trend and comparative analysis.
- **1.3 Health Metrics Analysis:** Compute health alerts, risk levels, and identify if a doctor visit is warranted.
- **1.4 Health Scoring:** Calculate a weighted health score with grades and recommendations.
- **1.5 Comparative Trend Analysis:** Compare current week data with the previous week to assess progress.
- **1.6 AI-Generated Reporting:** Use GPT-4o-mini to create detailed health reports and predictive health forecasts.
- **1.7 Alerting & Notifications:** Send emails, SMS, Slack messages, and schedule calendar events based on risk thresholds.
- **1.8 Caching & Device Sync:** Cache latest health scores in Redis and trigger wearable device sync APIs.
- **1.9 Logging & Reporting:** Log analytics to MongoDB and generate PDF reports via external service.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Normalization

- **Overview:**  
Receives incoming JSON health data via webhook, normalizes the raw data for consistency, and stores it in the primary database.

- **Nodes Involved:**  
  - Webhook - Health Data Input  
  - Normalize Health Data  
  - Store in Database  
  - Trigger Wearable Device Sync

- **Node Details:**

1. **Webhook - Health Data Input**  
   - Type: Webhook  
   - Role: Entry point receiving real-time health data payloads (e.g., from wearables or manual inputs).  
   - Config: Path set to "health-data". Accepts POST requests with JSON body.  
   - Inputs: External HTTP POST  
   - Outputs: Passes raw payload to normalization node.  
   - Edge Cases: Invalid payload, missing fields, large payloads, webhook downtime.

2. **Normalize Health Data**  
   - Type: Code (JavaScript)  
   - Role: Transforms incoming data into a consistent format, handles units and data validation.  
   - Config: Raw JavaScript code (not shown in JSON snippet) to standardize metrics like heart rate, blood pressure, temperature, etc.  
   - Inputs: Webhook output.  
   - Outputs: Normalized data to database storage and downstream nodes.  
   - Edge Cases: Data parsing errors, unexpected formats, missing/invalid values.

3. **Store in Database**  
   - Type: Postgres  
   - Role: Persist normalized health records into "health_records" table in PostgreSQL.  
   - Config: Table "health_records", schema "public".  
   - Inputs: Normalized health data.  
   - Outputs: Triggers logging node.  
   - Edge Cases: DB connection errors, duplicate entries, schema mismatches.

4. **Trigger Wearable Device Sync**  
   - Type: HTTP Request  
   - Role: Post-processing trigger to synchronize wearable device data via Fitbit API for the patient.  
   - Config: URL constructed using patient_id, POST method, OAuth2 Fitbit credentials.  
   - Inputs: Normalized data.  
   - Outputs: None used downstream.  
   - Edge Cases: API failures, auth token expiry, rate limits.

#### 1.2 Historical Data Retrieval

- **Overview:**  
Fetches recent health data history (last 30 records) for the patient to enable trend and comparative analyses.

- **Nodes Involved:**  
  - Get Recent History

- **Node Details:**

1. **Get Recent History**  
   - Type: Postgres (Execute Query)  
   - Role: Queries "health_records" table for the last 30 entries ordered by timestamp descending.  
   - Config: SQL query: `SELECT * FROM health_records ORDER BY timestamp DESC LIMIT 30`  
   - Inputs: Triggered after storing normalized data.  
   - Outputs: Provides historical dataset to health metrics analysis node.  
   - Edge Cases: DB errors, empty history for new patients.

#### 1.3 Health Metrics Analysis

- **Overview:**  
Analyzes current and historical health data to detect abnormalities, compute alerts, risk levels, and decide if a doctor visit is needed.

- **Nodes Involved:**  
  - Analyze Health Metrics  
  - Check If Doctor Visit Needed

- **Node Details:**

1. **Analyze Health Metrics**  
   - Type: Code (JavaScript)  
   - Role: Calculates averages over last 7 days, flags abnormal metrics (heart rate, BP, temperature, sleep, symptoms), sets risk level (normal, warning, critical), and compiles alerts.  
   - Key Logic:  
     - Heart rate abnormal if <60 or >100 bpm  
     - High BP if systolic >140 or diastolic >90  
     - Low BP if systolic <90 or diastolic <60  
     - Elevated temp > 38.0°C  
     - Sleep average < 6 hours triggers warning  
     - Symptoms other than "none" trigger warning  
     - Doctor visit needed if risk is critical or 3+ alerts  
   - Inputs: Current normalized data + recent history  
   - Outputs: JSON with alerts array, riskLevel, needsDoctorVisit boolean, averages, and trends.  
   - Edge Cases: Missing data fields, divide by zero if no historical data, inconsistent units.

2. **Check If Doctor Visit Needed**  
   - Type: If Conditional  
   - Role: Branches workflow based on `needsDoctorVisit` boolean from analysis.  
   - Config: Condition tests if `needsDoctorVisit === true`.  
   - Inputs: Output of Analyze Health Metrics.  
   - Outputs:  
     - True branch: Generates detailed health report.  
     - False branch: No action needed (noOp node).  
   - Edge Cases: Expression evaluation errors, null values.

#### 1.4 Health Scoring

- **Overview:**  
Computes an overall health score (0-100) based on weighted metrics, assigns grades A+ to F, and produces recommendations.

- **Nodes Involved:**  
  - Calculate Health Score

- **Node Details:**

1. **Calculate Health Score**  
   - Type: Code (JavaScript)  
   - Role: Applies scoring penalties for deviations in heart rate, blood pressure, temperature, sleep, activity (steps), and symptoms severity.  
   - Scoring Highlights:  
     - Heart rate deviation from 75 bpm reduces score up to 20 points  
     - High BP can reduce score by up to 25 points  
     - Temperature extremes reduce score up to 15 points  
     - Poor sleep and low activity reduce score up to 15 points each  
     - Symptoms severity reduces score by 5 or 10 points  
   - Determines letter grade and tailored health recommendation.  
   - Outputs: Enriched JSON with healthScore object including breakdown, grade, recommendation, and timestamp.  
   - Edge Cases: Missing numeric fields, invalid symptom descriptions.

#### 1.5 Comparative Trend Analysis

- **Overview:**  
Compares current week’s health averages against previous week to identify trends and insights.

- **Nodes Involved:**  
  - Compare with Previous Week

- **Node Details:**

1. **Compare with Previous Week**  
   - Type: Code (JavaScript)  
   - Role: Calculates previous week's averages and compares them to current week’s heart rate and sleep data.  
   - Outputs trend classification ("improving" or "needs attention") and insight messages if significant changes (>10% HR, >15% sleep).  
   - Returns updated dataset with comparison results.  
   - Edge Cases: Insufficient historical data (returns `available: false`).

#### 1.6 AI-Generated Reporting

- **Overview:**  
Generates detailed personalized health reports and predictive health forecasts using GPT-4o-mini AI models.

- **Nodes Involved:**  
  - Generate Health Report  
  - AI Predictive Health Agent

- **Node Details:**

1. **Generate Health Report**  
   - Type: Langchain Agent (AI Model)  
   - Role: Uses patient data, current vitals, lifestyle metrics, weekly comparisons, and health score breakdown to generate a multi-section detailed health report with recommendations and risk assessments.  
   - Prompt includes explicit report structure and compassionate medical language.  
   - Inputs: Health data and analysis JSON.  
   - Outputs: Textual health report for email and logging.  
   - Edge Cases: API errors, rate limits, incomplete input data.

2. **AI Predictive Health Agent**  
   - Type: Langchain Agent (AI Model)  
   - Role: Produces a 30-day health trajectory forecast, risk prediction, lifestyle intervention suggestions, and mental health considerations based on trends and alerts.  
   - Uses system prompt emphasizing evidence-based, actionable advice.  
   - Inputs: Current data with comparative trends and health score.  
   - Outputs: Predictive health insights text.  
   - Edge Cases: Similar to above; depends on AI service availability.

#### 1.7 Alerting & Notifications

- **Overview:**  
Sends emails, SMS, Slack alerts, and schedules calendar events based on critical health risk detection.

- **Nodes Involved:**  
  - Send Health Report Email  
  - Send Urgent Doctor Alert  
  - Emergency Contact SMS Alert  
  - Slack Alert to Care Team  
  - Schedule Follow-up Calendar Event  
  - No Action Needed (noOp node)

- **Node Details:**

1. **Send Health Report Email**  
   - Type: Email Send  
   - Role: Sends comprehensive health report email to patient with summary, AI-generated analysis, and quick vitals.  
   - Subject dynamically indicates if doctor visit is recommended.  
   - Inputs: Generated health report and current data.  
   - Outputs: Triggers urgent alert email if needed.  
   - Edge Cases: SMTP failures, invalid email addresses.

2. **Send Urgent Doctor Alert**  
   - Type: Email Send  
   - Role: Sends critical alert email when doctor visit is needed, listing alerts and next steps.  
   - Inputs: Health score and alerts data.  
   - Edge Cases: Same as above.

3. **Emergency Contact SMS Alert**  
   - Type: Twilio SMS  
   - Role: Sends urgent SMS to emergency contact phone number with critical alerts and recommendation.  
   - Inputs: Analysis alerts, risk level, patient name, report URL.  
   - Edge Cases: Missing emergency phone number, Twilio API errors.

4. **Slack Alert to Care Team**  
   - Type: Slack Post Message  
   - Role: Posts critical health alert summary to a configured Slack channel for care team awareness.  
   - Inputs: Patient info, health score, risk level, alerts, recommendations.  
   - Edge Cases: Slack webhook failures, throttling.

5. **Schedule Follow-up Calendar Event**  
   - Type: Google Calendar Create Event  
   - Role: Creates a follow-up appointment event 1 day after alert with 1-hour duration in configured calendar.  
   - Inputs: Current date/time for scheduling.  
   - Edge Cases: Calendar API permissions, invalid calendar ID.

6. **No Action Needed**  
   - Type: No Operation  
   - Role: Placeholder node for cases where no doctor visit is required.  
   - Inputs: Condition false branch.  
   - Outputs: Ends flow without further action.

#### 1.8 Caching & Device Sync

- **Overview:**  
Caches latest health score data in Redis for quick access and triggers wearable device sync.

- **Nodes Involved:**  
  - Store Health Score in Redis Cache  
  - Trigger Wearable Device Sync (already covered in 1.1)

- **Node Details:**

1. **Store Health Score in Redis Cache**  
   - Type: Redis  
   - Role: Stores serialized health score with expiry (7 days) using patient ID as key.  
   - Inputs: Calculated health score JSON.  
   - Edge Cases: Redis connection issues.

#### 1.9 Logging & Reporting

- **Overview:**  
Logs comprehensive analytics to MongoDB and generates PDF reports via external API.

- **Nodes Involved:**  
  - Log to MongoDB Health History  
  - Generate PDF Report

- **Node Details:**

1. **Log to MongoDB Health History**  
   - Type: MongoDB Insert  
   - Role: Inserts document with patient_id, timestamp, metrics, analysis, AI report, and actions into "health_analytics" collection.  
   - Inputs: Normalized data and analysis.  
   - Edge Cases: DB connection failures.

2. **Generate PDF Report**  
   - Type: HTTP Request  
   - Role: Calls external PDF generation API (PDFMonkey) with patient name, health score, and analysis payload to generate a formatted PDF report.  
   - Inputs: Current data and analysis.  
   - Edge Cases: API key missing, request failures.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                               | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                                   |
|--------------------------------|----------------------------------|-----------------------------------------------|-------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook - Health Data Input     | Webhook                          | Receives incoming health data                  | External HTTP                       | Normalize Health Data                   | ## How It Works: System collects real-time wearable data, normalizes, analyzes trends, alerts, and follow-up. |
| Normalize Health Data           | Code                             | Standardizes raw input data                     | Webhook - Health Data Input         | Store in Database, Get Recent History, Trigger Wearable Device Sync |                                                                                                               |
| Store in Database               | Postgres                        | Persists normalized records                     | Normalize Health Data               | Log to MongoDB Health History           |                                                                                                               |
| Trigger Wearable Device Sync    | HTTP Request                    | Triggers external wearable data sync            | Normalize Health Data               | None                                   |                                                                                                               |
| Log to MongoDB Health History   | MongoDB Insert                 | Logs analytics and AI reports                    | Store in Database                  | None                                   |                                                                                                               |
| Get Recent History              | Postgres (Execute Query)         | Retrieves last 30 health records for trend      | Normalize Health Data               | Analyze Health Metrics                  |                                                                                                               |
| Analyze Health Metrics          | Code                             | Computes alerts, risk level, doctor visit need  | Get Recent History                 | Check If Doctor Visit Needed, Calculate Health Score |                                                                                                               |
| Check If Doctor Visit Needed    | If                              | Branches based on doctor visit necessity        | Analyze Health Metrics             | Generate Health Report, No Action Needed |                                                                                                               |
| Generate Health Report          | Langchain Agent                  | Generates detailed personalized health report  | Check If Doctor Visit Needed       | Send Health Report Email                |                                                                                                               |
| Send Health Report Email        | Email Send                      | Emails detailed health report to patient        | Generate Health Report             | Send Urgent Doctor Alert                |                                                                                                               |
| Send Urgent Doctor Alert        | Email Send                      | Sends critical alert email if urgent            | Send Health Report Email           | None                                   |                                                                                                               |
| Emergency Contact SMS Alert     | Twilio SMS                     | Sends SMS alert to emergency contact             | Check Critical Threshold           | None                                   |                                                                                                               |
| Slack Alert to Care Team        | Slack Post Message             | Posts critical alerts to care team Slack channel| Check Critical Threshold           | None                                   |                                                                                                               |
| Schedule Follow-up Calendar Event | Google Calendar Create Event    | Schedules follow-up appointment                   | Check Critical Threshold           | None                                   |                                                                                                               |
| No Action Needed               | No Operation                    | Ends flow if no doctor visit needed              | Check If Doctor Visit Needed (false branch) | None                                   |                                                                                                               |
| Calculate Health Score          | Code                             | Calculates weighted health score and grading     | Analyze Health Metrics             | Compare with Previous Week, Check Critical Threshold, Store Health Score in Redis Cache |                                                                                                               |
| Compare with Previous Week      | Code                             | Compares current week with previous week trends  | Calculate Health Score             | Check If Doctor Visit Needed, AI Predictive Health Agent, Generate PDF Report |                                                                                                               |
| AI Predictive Health Agent      | Langchain Agent                  | Provides 30-day health forecast and interventions| Compare with Previous Week         | None                                   |                                                                                                               |
| Generate PDF Report             | HTTP Request                    | Generates PDF health report via external API     | Compare with Previous Week         | None                                   |                                                                                                               |
| Check Critical Threshold        | If                              | Detects critical health score and triggers alerts| Calculate Health Score             | Emergency Contact SMS Alert, Slack Alert to Care Team, Schedule Follow-up Calendar Event |                                                                                                               |
| Store Health Score in Redis Cache | Redis                          | Caches latest health score data                   | Calculate Health Score             | None                                   |                                                                                                               |
| OpenAI GPT-4                   | Langchain LM Chat OpenAI        | AI model for language generation                  | Generate Health Report             | None                                   |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `health-data`  
   - Accept POST requests with JSON body.

2. **Add Code Node "Normalize Health Data"**  
   - Purpose: Standardize incoming health data fields and units.  
   - Connect input from webhook node.  
   - Use JavaScript to parse and normalize fields such as heart_rate, bp_systolic, bp_diastolic, temperature, sleep_hours, etc.

3. **Add Postgres Node "Store in Database"**  
   - Operation: Insert into table `health_records` in schema `public`.  
   - Connect input from Normalize Health Data node.  
   - Configure database credentials accordingly.

4. **Add HTTP Request Node "Trigger Wearable Device Sync"**  
   - Method: POST  
   - URL: `https://api.fitbit.com/1/user/{{patient_id}}/activities/sync.json`  
   - Authentication: Fitbit OAuth2 credentials.  
   - Connect input from Normalize Health Data node.

5. **Add Postgres Node "Get Recent History"**  
   - Operation: Execute query  
   - Query: `SELECT * FROM health_records ORDER BY timestamp DESC LIMIT 30`  
   - Connect input from Normalize Health Data node.

6. **Add Code Node "Analyze Health Metrics"**  
   - Inputs: Combine current normalized data (first input) and historical data (all inputs).  
   - Logic: Calculate averages over last 7 days, detect abnormalities, set riskLevel, alerts, and needsDoctorVisit boolean.  
   - Connect input from Get Recent History node.

7. **Add If Node "Check If Doctor Visit Needed"**  
   - Condition: Check if `analysis.needsDoctorVisit` is true.  
   - Connect input from Analyze Health Metrics node.

8. **Add Code Node "Calculate Health Score"**  
   - Calculate weighted health score and assign grade and recommendation.  
   - Connect input from Analyze Health Metrics node.

9. **Add Code Node "Compare with Previous Week"**  
   - Compare current week averages with prior week.  
   - Connect input from Calculate Health Score node.

10. **Add Langchain Agent Node "Generate Health Report"**  
    - Prompt includes patient profile, current vitals, lifestyle, weekly comparison, and alerts.  
    - Connect input from Check If Doctor Visit Needed (true branch).

11. **Add Email Send Node "Send Health Report Email"**  
    - Sends report to patient email (fallback to default if missing).  
    - Connect input from Generate Health Report node.

12. **Add Email Send Node "Send Urgent Doctor Alert"**  
    - Sends urgent alert email when doctor visit recommended.  
    - Connect input from Send Health Report Email node.

13. **Add Twilio SMS Node "Emergency Contact SMS Alert"**  
    - Sends SMS alert to emergency contact if critical threshold met.  
    - Connect input from Check Critical Threshold node.

14. **Add Slack Node "Slack Alert to Care Team"**  
    - Posts critical health alerts to Slack.  
    - Connect input from Check Critical Threshold node.

15. **Add Google Calendar Node "Schedule Follow-up Calendar Event"**  
    - Schedule 1-day follow-up event with 1-hour duration.  
    - Connect input from Check Critical Threshold node.

16. **Add No Operation Node "No Action Needed"**  
    - Connect input from Check If Doctor Visit Needed (false branch).

17. **Add MongoDB Node "Log to MongoDB Health History"**  
    - Insert patient analytics and AI reports into `health_analytics` collection.  
    - Connect input from Store in Database node.

18. **Add HTTP Request Node "Generate PDF Report"**  
    - POST to external PDF generation API with patient name, health score, and analysis.  
    - Connect input from Compare with Previous Week node.

19. **Add Redis Node "Store Health Score in Redis Cache"**  
    - Store latest health score keyed by patient ID with 7-day TTL.  
    - Connect input from Calculate Health Score node.

20. **Add Langchain Agent Node "AI Predictive Health Agent"**  
    - Generate 30-day health forecast and lifestyle recommendations.  
    - Connect input from Compare with Previous Week node.

21. **Add OpenAI GPT-4 Node**  
    - Used as language model for Generate Health Report node.  
    - Provide OpenAI API credentials.

**Connect nodes according to the flow described in section 1 and 2** ensuring proper data passing and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| The system collects real-time wearable health data, normalizes it, and uses AI to analyze trends and risk scores. It detects anomalies by comparing with historical patterns and automatically triggers alerts and follow-up actions. | See Sticky Note at position [-240, 496] in the workflow.                                                           |
| Prerequisites include: Wearable API access, patient database, GPT-4 API key, SMTP for email, optional Slack and Twilio for alerts, and calendar integration for appointments.                                                  | See Sticky Note1 at position [1296, 496] in the workflow.                                                          |
| Use cases encompass monitoring diabetic glucose levels, elderly vital signs and fall risk, corporate wellness assessments, and post-surgery recovery monitoring.                                                              | See Sticky Note1.                                                                                                  |
| Customization options: Modify risk scoring algorithms, add new health metrics, and integrate telemedicine platforms for live consultations.                                                                                   | See Sticky Note1.                                                                                                  |
| Benefits include early intervention reducing hospital readmissions and automating up to 80% of routine health monitoring tasks.                                                                                              | See Sticky Note1.                                                                                                  |
| Workflow steps include ingestion → normalization → scoring → analysis → comparison → alerting → reporting → follow-up scheduling.                                                                                             | See Sticky Note2 at position [416, 496] in the workflow.                                                           |
| Setup instructions highlight configuring webhook, databases, GPT-4 API, notification channels, calendar, and alert thresholds.                                                                                                | See Sticky Note2.                                                                                                  |

---

**Disclaimer:**  
The content above is extracted and analyzed exclusively from an n8n automated workflow designed for health data monitoring. It complies with all content policies and handles only legal, public, and safe data. This system is an assistive tool and does not replace professional medical advice or diagnosis.