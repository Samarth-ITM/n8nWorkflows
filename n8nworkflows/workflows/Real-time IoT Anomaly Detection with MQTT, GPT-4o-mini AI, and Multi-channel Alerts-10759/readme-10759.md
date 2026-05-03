Real-time IoT Anomaly Detection with MQTT, GPT-4o-mini AI, and Multi-channel Alerts

https://n8nworkflows.xyz/workflows/real-time-iot-anomaly-detection-with-mqtt--gpt-4o-mini-ai--and-multi-channel-alerts-10759


# Real-time IoT Anomaly Detection with MQTT, GPT-4o-mini AI, and Multi-channel Alerts

### 1. Workflow Overview

This workflow implements a **Real-time IoT Anomaly Detection Pipeline** that ingests sensor data via MQTT, preprocesses and validates it, normalizes sensor readings, stores raw data, performs statistical anomaly detection, and then applies an AI agent (GPT-4o-mini) to validate and classify anomalies. Confirmed anomalies trigger multi-channel alerts via a dashboard API, Slack, and email notifications. Additionally, the workflow includes a scheduled ML model retraining process to maintain detection accuracy.

**Target Use Cases:**  
- Real-time monitoring of IoT sensor networks  
- Anomaly detection in industrial or environmental sensors  
- Automated incident alerting for operational teams  
- Predictive maintenance and fault detection  

**Logical Blocks:**  
- **1.1 Data Ingestion & Configuration:** MQTT trigger and workflow configuration parameters  
- **1.2 Edge Preprocessing & Validation:** Data quality checks and cleaning via custom code  
- **1.3 Data Normalization & Storage:** Standardizing sensor units and storing raw data in a time-series database  
- **1.4 Statistical Anomaly Detection:** Initial anomaly flagging using Z-score method  
- **1.5 AI Anomaly Detection Agent:** Context-aware validation and classification of anomalies with GPT-4o-mini  
- **1.6 Anomaly Handling & Alerts:** Storing anomaly records and sending multi-channel alerts (dashboard, Slack, email)  
- **1.7 ML Model Retraining:** Scheduled historical data fetching and model retraining via API  

---

### 2. Block-by-Block Analysis

#### 1.1 Data Ingestion & Configuration

**Overview:**  
This block receives real-time IoT sensor data via MQTT topics and sets essential workflow parameters such as anomaly thresholds, API endpoints, and alert destinations.

**Nodes Involved:**  
- MQTT Sensor Data Ingestion  
- Workflow Configuration

**Node Details:**

- **MQTT Sensor Data Ingestion**  
  - Type: MQTT Trigger  
  - Role: Subscribes to MQTT topics `sensors/+/data` to receive sensor data streams in real-time  
  - Configuration: Listens on wildcard topic to handle multiple sensors  
  - Input: External MQTT broker (requires broker credentials configured in n8n)  
  - Output: Raw sensor data payloads  
  - Failure Modes: Connection loss with MQTT broker, malformed payloads  
  - Notes: Lightweight, reliable ingestion for high-volume data streams

- **Workflow Configuration**  
  - Type: Set  
  - Role: Defines static workflow parameters used downstream (thresholds, API URLs, alert channels)  
  - Parameters include:  
    - anomalyThreshold = 2.5 (Z-score threshold for statistical detection)  
    - dashboardApiUrl = placeholder for dashboard endpoint  
    - slackChannel = placeholder for Slack channel  
    - alertEmail = placeholder for alert recipient email  
    - mlModelApiUrl = placeholder for ML model training API endpoint  
  - Input: MQTT Sensor Data Ingestion  
  - Output: Configuration parameters attached to the flow  
  - Failure Modes: Misconfiguration may prevent correct alert routing or model retraining  

---

#### 1.2 Edge Preprocessing & Validation

**Overview:**  
Validates incoming sensor data payloads, checking mandatory fields, data types, and logical correctness (e.g., timestamps, sensor IDs). Assigns data quality scores and prepares cleaned data for further processing.

**Nodes Involved:**  
- Edge Preprocessing & Validation

**Node Details:**

- **Edge Preprocessing & Validation**  
  - Type: Code (JavaScript)  
  - Role: Runs per item to validate sensor payloads and enhance data quality metrics  
  - Key Logic:  
    - Parses JSON payload if input is a string  
    - Checks for required fields: `sensor_id`, `timestamp`, `value`, `unit`  
    - Validates types and ranges (e.g., non-empty sensor_id, valid timestamp not too old or future, numeric value in reasonable range)  
    - Calculates a `dataQualityScore` (0-100) deducting points for warnings and missing optional fields  
    - Returns an object with validation status, errors, warnings, and cleaned data including processing timestamp  
  - Inputs: Workflow Configuration outputs + MQTT data  
  - Outputs: Validation result object with cleaned sensor data or error details  
  - Failure Modes: Invalid JSON, missing fields, invalid types cause data to be marked invalid and excluded from downstream  
  - Notes: Critical quality gate to avoid garbage data propagation  

---

#### 1.3 Data Normalization & Storage

**Overview:**  
Standardizes sensor readings (e.g., converts temperature from Fahrenheit to Celsius) and stores raw and normalized sensor readings into a PostgreSQL time-series database.

**Nodes Involved:**  
- Normalize Sensor Data  
- Store Raw Data in Time-Series DB  
- Aggregate Sensor Readings

**Node Details:**

- **Normalize Sensor Data**  
  - Type: Set  
  - Role: Creates normalized fields for consistent analysis  
  - Key Configuration:  
    - Converts temperature Fahrenheit to Celsius if sensor type is temperature and unit is ‘F’  
    - Passes through other key fields and adds ingestion timestamp  
  - Input: Edge Preprocessing & Validation output (cleaned data)  
  - Output: Normalized sensor data for storage and analysis  
  - Failure Modes: Incorrect sensor type or unit could cause wrong normalization

- **Store Raw Data in Time-Series DB**  
  - Type: PostgreSQL Node  
  - Role: Inserts sensor readings into `sensor_readings` table with schema `public`  
  - Columns: sensor_id, timestamp, value, unit, data_quality, ingestion_time, normalized_value  
  - Input: Normalize Sensor Data output  
  - Output: Confirms DB insertion  
  - Failure Modes: Database connection issues, data schema mismatch

- **Aggregate Sensor Readings**  
  - Type: Aggregate  
  - Role: Combines all stored readings for statistical processing  
  - Input: Store Raw Data in Time-Series DB output  
  - Output: Aggregated data array for anomaly detection  
  - Failure Modes: Empty data sets, aggregation errors

---

#### 1.4 Statistical Anomaly Detection

**Overview:**  
Performs a fast initial anomaly detection step using the Z-score method on aggregated sensor readings to flag potential outliers.

**Nodes Involved:**  
- Statistical Anomaly Detection

**Node Details:**

- **Statistical Anomaly Detection**  
  - Type: Code (JavaScript)  
  - Role: Calculates mean, standard deviation, and Z-scores of values; flags anomalies exceeding threshold  
  - Configured Threshold: Uses `anomalyThreshold` from workflow config (default 3)  
  - Adds fields per item: `anomaly_score`, `is_anomaly`, and detailed statistical analysis  
  - Input: Aggregate Sensor Readings output (array of sensor values)  
  - Output: Enriched items with statistical anomaly flags  
  - Failure Modes: Division by zero if stdDev = 0 handled gracefully; insufficient data points can reduce accuracy  
  - Notes: Quick filtering to reduce load on AI agent  

---

#### 1.5 AI Anomaly Detection Agent

**Overview:**  
Uses GPT-4o-mini to analyze statistical anomaly flags with contextual understanding, reducing false positives and classifying anomaly types with recommendations.

**Nodes Involved:**  
- AI Anomaly Detection Agent  
- OpenAI Chat Model  
- Check for Anomalies

**Node Details:**

- **AI Anomaly Detection Agent**  
  - Type: Langchain Agent Node  
  - Role: Acts as IoT anomaly expert analyzing sensor data and statistical metrics  
  - Prompt: Requests JSON response with `is_anomaly`, `confidence`, `anomaly_type`, `explanation`, `recommended_action`  
  - Input: Statistical Anomaly Detection output  
  - Output: AI-validated anomaly assessment  
  - Failure Modes: API rate limits, model errors, prompt misconfigurations

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI chat model node  
  - Role: Backend AI model `gpt-4o-mini` powering the agent  
  - Credentials: Requires OpenAI API key set in n8n credentials  
  - Input: Connected from AI Anomaly Detection Agent (language model)  
  - Output: AI-generated anomaly analysis

- **Check for Anomalies**  
  - Type: If  
  - Role: Conditional branching based on AI agent’s `is_anomaly` boolean output  
  - Input: AI Anomaly Detection Agent output  
  - Output: Routes workflow to anomaly handling if true; otherwise terminates or continues silently  
  - Failure Modes: Expression errors if AI output malformed

---

#### 1.6 Anomaly Handling & Alerts

**Overview:**  
Stores confirmed anomaly records, sends anomaly data to monitoring dashboards, and alerts stakeholders through Slack and email.

**Nodes Involved:**  
- Store Anomaly Records  
- Send Alert to Dashboard API  
- Send Slack Alert  
- Send Email Alert

**Node Details:**

- **Store Anomaly Records**  
  - Type: PostgreSQL Node  
  - Role: Inserts anomaly details into `anomaly_records` table in `public` schema  
  - Columns: sensor_id, timestamp, confidence, detected_at, explanation, anomaly_type, recommended_action  
  - Input: Check for Anomalies output (true branch)  
  - Output: Confirms insertion into anomaly database  
  - Failure Modes: DB connectivity, schema mismatch

- **Send Alert to Dashboard API**  
  - Type: HTTP Request  
  - Role: Posts anomaly details to configured monitoring dashboard endpoint  
  - HTTP Method: POST  
  - Headers: Content-Type application/json  
  - Body: Sends sensor_id, timestamp, anomaly_type, confidence, explanation  
  - Input: Store Anomaly Records output  
  - Output: HTTP response from dashboard API  
  - Failure Modes: Network errors, API authentication issues, endpoint unavailability

- **Send Slack Alert**  
  - Type: Slack Node  
  - Role: Sends formatted alert message to Slack channel configured in Workflow Configuration  
  - Message includes sensor ID, anomaly type, confidence %, recommended action, and timestamp  
  - Authentication: OAuth2 with Slack credentials  
  - Input: Send Alert to Dashboard API output  
  - Output: Confirmation of Slack message sent  
  - Failure Modes: Slack API limits, invalid channel IDs, OAuth token expiry

- **Send Email Alert**  
  - Type: Email Send  
  - Role: Sends styled HTML email alert with anomaly details and recommended actions to configured email recipient  
  - Subject: “IoT Anomaly Alert: High Confidence Detection”  
  - From: noreply@iot-monitoring.com  
  - Input: Send Slack Alert output  
  - Output: Email delivery confirmation  
  - Failure Modes: SMTP server issues, invalid email addresses

---

#### 1.7 ML Model Retraining

**Overview:**  
Runs on a weekly schedule to fetch recent historical sensor data and trigger ML model retraining via an external API.

**Nodes Involved:**  
- Model Retraining Schedule  
- Fetch Historical Data for Retraining  
- Retrain ML Model via API

**Node Details:**

- **Model Retraining Schedule**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow every 7 days at 2 AM  
  - Input: None (time-based)  
  - Output: Trigger signal for retraining flow  
  - Failure Modes: Scheduler downtime (rare)

- **Fetch Historical Data for Retraining**  
  - Type: PostgreSQL Node  
  - Role: Queries last 30 days of sensor readings from `sensor_readings` table  
  - SQL Query: Selects sensor_id, timestamp, value, normalized_value, data_quality ordered descending by timestamp  
  - Input: Model Retraining Schedule output  
  - Output: Dataset for retraining

- **Retrain ML Model via API**  
  - Type: HTTP Request  
  - Role: Posts training data and parameters (epochs, batch size, learning rate) to ML model training API endpoint  
  - HTTP Method: POST, Content-Type application/json  
  - Body: JSON containing `"training_data"` and `"parameters"`  
  - Input: Fetch Historical Data for Retraining output  
  - Output: API response confirming retraining job  
  - Failure Modes: API downtime, invalid training data format, authentication errors

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                    | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                 |
|--------------------------------|----------------------------|----------------------------------|--------------------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| MQTT Sensor Data Ingestion      | MQTT Trigger               | Ingest real-time sensor data     | -                              | Workflow Configuration         | MQTT ingests real-time sensor data from connected devices. Lightweight and reliable.        |
| Workflow Configuration          | Set                        | Define workflow parameters       | MQTT Sensor Data Ingestion      | Edge Preprocessing & Validation | Setup Steps: Configure MQTT, ML model, AI agent, alerts.                                    |
| Edge Preprocessing & Validation | Code                       | Validate and clean sensor data   | Workflow Configuration          | Normalize Sensor Data          | Critical quality gate to ensure valid data.                                                 |
| Normalize Sensor Data           | Set                        | Normalize units and fields       | Edge Preprocessing & Validation | Store Raw Data in Time-Series DB | Normalizes sensor readings for consistent AI interpretation.                                |
| Store Raw Data in Time-Series DB| PostgreSQL                 | Store sensor data                | Normalize Sensor Data           | Aggregate Sensor Readings      | Stores raw and normalized data in time-series DB.                                           |
| Aggregate Sensor Readings       | Aggregate                  | Aggregate data for analysis      | Store Raw Data in Time-Series DB| Statistical Anomaly Detection  | Statistical anomaly detection uses deviation, moving averages, rate-of-change.             |
| Statistical Anomaly Detection   | Code                       | Flag anomalies statistically    | Aggregate Sensor Readings       | AI Anomaly Detection Agent     | Fast first-pass detection of spikes with minimal compute.                                   |
| AI Anomaly Detection Agent      | Langchain Agent            | Contextual AI anomaly validation| Statistical Anomaly Detection   | Check for Anomalies            | AI agent reduces noise by distinguishing real issues.                                      |
| OpenAI Chat Model               | Langchain OpenAI Chat Model| AI language model                | AI Anomaly Detection Agent      | AI Anomaly Detection Agent     | Uses GPT-4o-mini. Requires OpenAI credentials.                                              |
| Check for Anomalies             | If                         | Branch on anomaly detection      | AI Anomaly Detection Agent      | Store Anomaly Records          | Routes confirmed anomalies for alerting and storage.                                       |
| Store Anomaly Records           | PostgreSQL                 | Store anomaly records            | Check for Anomalies             | Send Alert to Dashboard API    | Stores verified anomaly details in DB.                                                     |
| Send Alert to Dashboard API     | HTTP Request               | Notify monitoring dashboard      | Store Anomaly Records           | Send Slack Alert               | Publishes verified anomalies with context and severity.                                    |
| Send Slack Alert                | Slack                      | Send Slack notifications         | Send Alert to Dashboard API     | Send Email Alert               | Sends immediate alerts to Slack channel.                                                   |
| Send Email Alert                | Email Send                 | Send email notifications         | Send Slack Alert                | -                             | Sends detailed HTML email alert with recommended actions.                                  |
| Model Retraining Schedule       | Schedule Trigger           | Weekly retraining trigger        | -                              | Fetch Historical Data for Retraining | Retrains models periodically to maintain accuracy.                                        |
| Fetch Historical Data for Retraining | PostgreSQL            | Retrieve recent sensor data      | Model Retraining Schedule       | Retrain ML Model via API       | Fetches last 30 days data for model retraining.                                            |
| Retrain ML Model via API        | HTTP Request               | Trigger ML model retraining      | Fetch Historical Data for Retraining | -                         | Posts training data and parameters to ML API endpoint.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MQTT Sensor Data Ingestion node**  
   - Type: MQTT Trigger  
   - Configure MQTT broker credentials  
   - Set topic to `sensors/+/data` to subscribe to all sensor data topics  

2. **Create Workflow Configuration node**  
   - Type: Set  
   - Add variables:  
     - `anomalyThreshold` (Number): 2.5  
     - `dashboardApiUrl` (String): Set your dashboard API endpoint URL  
     - `slackChannel` (String): Set Slack channel name or ID for alerts  
     - `alertEmail` (String): Set email address for anomaly alerts  
     - `mlModelApiUrl` (String): Set ML model retraining API endpoint URL  
   - Connect MQTT Sensor Data Ingestion → Workflow Configuration  

3. **Create Edge Preprocessing & Validation node**  
   - Type: Code (JavaScript)  
   - Paste the provided JS validation and preprocessing script  
   - Set mode to "runOnceForEachItem"  
   - Connect Workflow Configuration → Edge Preprocessing & Validation  

4. **Create Normalize Sensor Data node**  
   - Type: Set  
   - Configure fields:  
     - `sensor_id`: `={{ $json.sensor_id }}`  
     - `timestamp`: `={{ new Date($json.timestamp).toISOString() }}`  
     - `value`: `={{ parseFloat($json.value) }}`  
     - `unit`: `={{ $json.unit }}`  
     - `normalized_value`: `={{ $json.sensor_type === 'temperature' && $json.unit === 'F' ? (parseFloat($json.value) - 32) * 5/9 : parseFloat($json.value) }}`  
     - `data_quality`: `={{ $json.data_quality }}`  
     - `ingestion_time`: `={{ new Date().toISOString() }}`  
   - Connect Edge Preprocessing & Validation → Normalize Sensor Data  

5. **Create Store Raw Data in Time-Series DB node**  
   - Type: PostgreSQL  
   - Configure connection credentials to your PostgreSQL DB  
   - Table: `sensor_readings`, Schema: `public`  
   - Map columns: sensor_id, timestamp, value, unit, data_quality, ingestion_time, normalized_value from JSON inputs  
   - Connect Normalize Sensor Data → Store Raw Data in Time-Series DB  

6. **Create Aggregate Sensor Readings node**  
   - Type: Aggregate  
   - Configure to aggregate all input items (default settings)  
   - Connect Store Raw Data in Time-Series DB → Aggregate Sensor Readings  

7. **Create Statistical Anomaly Detection node**  
   - Type: Code (JavaScript)  
   - Paste provided JS code calculating mean, stdDev, Z-score, flagging anomalies  
   - Use `anomalyThreshold` from Workflow Configuration or default 3  
   - Connect Aggregate Sensor Readings → Statistical Anomaly Detection  

8. **Create AI Anomaly Detection Agent node**  
   - Type: Langchain Agent  
   - Set prompt describing IoT anomaly detection expert with expected JSON output structure  
   - Connect Statistical Anomaly Detection → AI Anomaly Detection Agent  

9. **Create OpenAI Chat Model node**  
   - Type: Langchain OpenAI Chat Model  
   - Select model `gpt-4o-mini`  
   - Configure OpenAI credentials (API key)  
   - Connect AI Anomaly Detection Agent (ai_languageModel input) → OpenAI Chat Model  

10. **Create Check for Anomalies node**  
    - Type: If  
    - Condition: Expression equals `true` for `$('AI Anomaly Detection Agent').item.json.is_anomaly`  
    - Connect AI Anomaly Detection Agent → Check for Anomalies  

11. **Create Store Anomaly Records node**  
    - Type: PostgreSQL  
    - Table: `anomaly_records`, Schema: `public`  
    - Map columns: sensor_id, timestamp, confidence, detected_at, explanation, anomaly_type, recommended_action from AI agent JSON output  
    - Connect Check for Anomalies (true branch) → Store Anomaly Records  

12. **Create Send Alert to Dashboard API node**  
    - Type: HTTP Request  
    - URL: `={{ $('Workflow Configuration').first().json.dashboardApiUrl }}`  
    - Method: POST  
    - Headers: Content-Type application/json  
    - Body parameters: sensor_id, timestamp, anomaly_type, confidence, explanation from JSON  
    - Connect Store Anomaly Records → Send Alert to Dashboard API  

13. **Create Send Slack Alert node**  
    - Type: Slack  
    - Message text: formatted with sensor ID, anomaly type, confidence, recommended action, timestamp (use expressions referencing AI Anomaly Detection Agent output)  
    - Channel ID: from `Workflow Configuration` node slackChannel variable  
    - Authentication: OAuth2 with Slack credentials  
    - Connect Send Alert to Dashboard API → Send Slack Alert  

14. **Create Send Email Alert node**  
    - Type: Email Send  
    - To: `={{ $('Workflow Configuration').first().json.alertEmail }}`  
    - From: noreply@iot-monitoring.com  
    - Subject: "IoT Anomaly Alert: High Confidence Detection"  
    - HTML Body: Use provided styled HTML template with anomaly details and recommended actions (expressions referencing AI Anomaly Detection Agent output)  
    - Connect Send Slack Alert → Send Email Alert  

15. **Create Model Retraining Schedule node**  
    - Type: Schedule Trigger  
    - Configure to trigger every 7 days at 2 AM  

16. **Create Fetch Historical Data for Retraining node**  
    - Type: PostgreSQL  
    - Query:  
      ```sql
      SELECT sensor_id, timestamp, value, normalized_value, data_quality 
      FROM sensor_readings 
      WHERE timestamp > NOW() - INTERVAL '30 days' 
      ORDER BY timestamp DESC
      ```  
    - Connect Model Retraining Schedule → Fetch Historical Data for Retraining  

17. **Create Retrain ML Model via API node**  
    - Type: HTTP Request  
    - URL: `={{ $('Workflow Configuration').first().json.mlModelApiUrl }}`  
    - Method: POST  
    - Headers: Content-Type application/json  
    - JSON Body:  
      ```json
      {
        "training_data": "={{ $('Fetch Historical Data for Retraining').all() }}",
        "parameters": {
          "epochs": 100,
          "batch_size": 32,
          "learning_rate": 0.001
        }
      }
      ```  
    - Connect Fetch Historical Data for Retraining → Retrain ML Model via API  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| MQTT provides lightweight, reliable ingestion for high-volume sensor data streams enabling near real-time anomaly detection. | Sticky Note: MQTT Ingests Sensor Stream |
| Workflow supports customization including threshold tuning, ML model swapping, notification channel modifications, and validation rules customization. | Sticky Note: Customization & Benefits |
| AI agent reduces false positives by contextualizing statistical anomalies using sensor correlations and domain knowledge. | Sticky Note: AI Anomaly Agent |
| Scheduled retraining ensures model adapts to sensor behavior changes over time, maintaining detection accuracy. | Sticky Note: Schedule ML Retraining |
| Alerts are sent via centralized dashboards, Slack, and email with actionable details to accelerate incident response. | Sticky Note: Notifications & Monitoring Dashboard |
| Project credits and setup instructions are summarized in setup sticky notes for easy onboarding. | Sticky Notes: Setup Steps, How It Works |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. All data processed is legal and public, respecting content policies and without inclusion of illegal or offensive content.