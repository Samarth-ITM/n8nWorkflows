Fraudulent Booking Detector: Identify Suspicious Travel Transactions with Google Gemini

https://n8nworkflows.xyz/workflows/fraudulent-booking-detector--identify-suspicious-travel-transactions-with-google-gemini-6266


# Fraudulent Booking Detector: Identify Suspicious Travel Transactions with Google Gemini

### 1. Workflow Overview

This workflow, titled **Fraudulent Booking Detector: Identify Suspicious Travel Transactions with Google Gemini**, is designed to detect potentially fraudulent travel booking transactions by analyzing incoming booking data with a combination of AI-powered risk assessment and rule-based checks. It targets use cases where travel booking platforms or payment processors want to automatically evaluate transactions for fraud indicators, assign risk scores, and trigger appropriate automated actions such as blocking users, flagging transactions for manual review, or simply logging the data for monitoring.

The workflow is logically divided into five main blocks:

- **1.1 Input Reception & Data Extraction**: Receives incoming booking data via webhook and extracts relevant fields for processing.
- **1.2 IP Geolocation & AI Analysis**: Enriches transaction data with geolocation info and leverages an AI agent powered by Google Gemini Chat model to analyze fraud risk.
- **1.3 Risk Calculation & Decision Logic**: Combines AI results with custom rule-based risk factors to compute a final risk score and level, branching logic based on risk thresholds.
- **1.4 Automated Actions & Notifications**: Takes automated actions such as blocking accounts or flagging transactions, and sends notification emails depending on risk severity.
- **1.5 Logging & Response**: Logs all transaction and risk data to Google Sheets and sends a structured response back to the caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Extraction

**Overview:**  
This block initiates the workflow by receiving booking transaction data via an HTTP POST webhook and extracts key details into a structured format for downstream processing.

**Nodes Involved:**  
- Booking Transaction Webhook  
- Extract Booking Data

**Node Details:**

- **Booking Transaction Webhook**  
  - *Type:* Webhook (HTTP Listener)  
  - *Role:* Entry point accepting POST requests at `/fraud-detection` path.  
  - *Configuration:* Listens for HTTP POST, responds via downstream Respond node.  
  - *Inputs:* External HTTP POST request with booking JSON payload.  
  - *Outputs:* Raw JSON containing booking transaction data.  
  - *Failures:* Network connectivity issues, invalid HTTP method calls, malformed JSON payloads.

- **Extract Booking Data**  
  - *Type:* Set node  
  - *Role:* Extracts and normalizes key booking fields for easy access: user_id, booking_amount, booking_time, ip_address, payment_method, booking_location, session_id.  
  - *Configuration:* Uses expressions to map from webhook body JSON fields.  
  - *Inputs:* Output of Booking Transaction Webhook.  
  - *Outputs:* Structured JSON with normalized booking data fields.  
  - *Failures:* Missing or malformed fields in incoming payload lead to empty or invalid extracted values.

---

#### 2.2 IP Geolocation & AI Analysis

**Overview:**  
This block enriches the extracted booking data with IP geolocation information and leverages an AI agent utilizing the Google Gemini Chat model to analyze the transaction for fraud risk.

**Nodes Involved:**  
- IP Geolocation Check  
- Google Gemini Chat Model  
- AI Agent

**Node Details:**

- **IP Geolocation Check**  
  - *Type:* HTTP Request  
  - *Role:* Queries external IP geolocation API (`ip-api.com`) to retrieve location details for the booking’s IP address.  
  - *Configuration:* Dynamic URL built with extracted IP address; no authentication.  
  - *Inputs:* Extract Booking Data node output for IP address.  
  - *Outputs:* JSON with city, country, proxy status, and geolocation status.  
  - *Failures:* API rate limits, IP lookup failures, network errors lead to missing or failed geolocation data.

- **Google Gemini Chat Model**  
  - *Type:* AI Language Model (Langchain integration)  
  - *Role:* Provides advanced AI language model capabilities for fraud detection analysis.  
  - *Configuration:* Uses "gemini-2.5-pro" model from Google PaLM API. Requires Google Palm API credentials.  
  - *Inputs:* Text prompts from AI Agent node.  
  - *Outputs:* AI-generated text output including risk analysis in JSON format.  
  - *Failures:* API quota limits, authentication errors, model unavailability.

- **AI Agent**  
  - *Type:* Langchain Agent node  
  - *Role:* Constructs a detailed prompt combining booking data and geolocation info, requests fraud risk analysis in specified JSON format from Google Gemini Chat Model.  
  - *Configuration:* Prompt instructs AI on analysis framework and response format; uses system message defining domain expertise and scoring guidelines.  
  - *Inputs:* Booking data, geolocation data, and AI model node output.  
  - *Outputs:* Text output containing structured JSON with risk_score, risk_level, reasons, fraud_indicators, recommendation.  
  - *Failures:* Parsing errors if AI output is malformed, invalid JSON extraction, AI model errors.

---

#### 2.3 Risk Calculation & Decision Logic

**Overview:**  
This block combines AI-generated risk scores with additional rule-based risk factors such as transaction amount, booking time, payment method, and geolocation results to compute a final risk score and level. It then routes the workflow based on critical or high-risk checks.

**Nodes Involved:**  
- Enhanced Risk Calculator  
- Critical Risk Check  
- High Risk Check

**Node Details:**

- **Enhanced Risk Calculator**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses AI agent JSON response, applies custom business rules to add/subtract risk points, calculates combined risk score and final risk level, compiles risk reasons and fraud indicators.  
  - *Configuration:* Includes fallback logic if AI JSON parsing fails; risk scoring thresholds for CRITICAL, HIGH, MEDIUM, LOW.  
  - *Inputs:* Booking data, geolocation data, AI Agent output.  
  - *Outputs:* Object containing user_id, booking_amount, combined risk_score, risk_level, recommendation, detailed risk factors, fraud indicators, location info, timestamps.  
  - *Failures:* Syntax errors in code, unexpected data formats, missing AI output, edge case handling for unknown IP geolocation or payment methods.

- **Critical Risk Check**  
  - *Type:* If node  
  - *Role:* Checks if final risk_level equals "CRITICAL".  
  - *Inputs:* Output of Enhanced Risk Calculator.  
  - *Outputs:* Branches workflow for critical risk actions or continues to next check.  
  - *Failures:* Unexpected or missing risk_level value.

- **High Risk Check**  
  - *Type:* If node  
  - *Role:* Checks if final risk_level equals "HIGH".  
  - *Inputs:* Output of Enhanced Risk Calculator.  
  - *Outputs:* Branches workflow for high risk actions or continues to logging.  
  - *Failures:* Same as above.

---

#### 2.4 Automated Actions & Notifications

**Overview:**  
Based on risk level, this block executes automated actions such as blocking user accounts, flagging transactions for review, and sending alert emails to designated recipients.

**Nodes Involved:**  
- Block User Account  
- Send a message (Critical)  
- Flag for Review  
- Send a message1 (High Risk)

**Node Details:**

- **Block User Account**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to external fraud management API to block user accounts flagged as CRITICAL risk.  
  - *Configuration:* Sends user_id in request body. Uses no authentication but error handling set to "continueRegularOutput" to avoid workflow failure.  
  - *Inputs:* Critical Risk Check true branch.  
  - *Outputs:* External API response.  
  - *Failures:* API unavailability, network issues.

- **Send a message** (Critical)  
  - *Type:* Gmail node  
  - *Role:* Sends detailed email alert for critical fraud cases to specified email address.  
  - *Configuration:* Uses OAuth2 Gmail credentials; email includes transaction details, risk factors, fraud indicators, and immediate action warning.  
  - *Inputs:* Critical Risk Check true branch.  
  - *Outputs:* Email sent confirmation.  
  - *Failures:* Email sending failure, credential expiration.

- **Flag for Review**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to external system to flag transactions for manual review for HIGH risk cases.  
  - *Configuration:* Sends user_id in request body, error handling similar to block user node.  
  - *Inputs:* High Risk Check true branch.  
  - *Outputs:* API response.  
  - *Failures:* Same as Block User Account node.

- **Send a message1** (High Risk)  
  - *Type:* Gmail node  
  - *Role:* Sends notification email for high-risk bookings, summarizing risk factors and recommended actions for review.  
  - *Configuration:* OAuth2 Gmail credentials, styled HTML email.  
  - *Inputs:* High Risk Check true branch.  
  - *Outputs:* Email confirmation.  
  - *Failures:* Same as above.

---

#### 2.5 Logging & Response

**Overview:**  
This final block logs all relevant transaction and risk data to Google Sheets for audit and monitoring and sends a structured JSON response back to the original webhook caller summarizing the analysis and actions taken.

**Nodes Involved:**  
- Log to Google Sheets  
- Send Response

**Node Details:**

- **Log to Google Sheets**  
  - *Type:* Google Sheets node  
  - *Role:* Appends transaction data, risk scores, indicators, and recommendations to a dedicated Google Sheet for record-keeping.  
  - *Configuration:* Uses service account credentials mapped to spreadsheet columns; automatic input data mapping enabled.  
  - *Inputs:* Output from Enhanced Risk Calculator.  
  - *Outputs:* Google Sheets append confirmation.  
  - *Failures:* Authentication errors, API quota limits.

- **Send Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends JSON response back to upstream caller indicating transaction processing status, risk level, score, recommendation, and actions taken.  
  - *Configuration:* Dynamically builds response body using expressions referencing risk calculator outputs and extracted session ID.  
  - *Inputs:* Output of Log to Google Sheets.  
  - *Outputs:* HTTP JSON response to original POST request.  
  - *Failures:* Response sending errors, timeout if workflow execution exceeds webhook timeout limits.

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                          | Input Node(s)                | Output Node(s)                        | Sticky Note                                                                                      |
|----------------------------|--------------------------------------------|----------------------------------------|-----------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| Booking Transaction Webhook | Webhook                                    | Entry point, receive booking data      | -                           | Extract Booking Data                | ## 1. Initial Data Ingestion & Extraction                                                     |
| Extract Booking Data        | Set                                        | Extract and normalize booking fields   | Booking Transaction Webhook  | IP Geolocation Check                | ## 1. Initial Data Ingestion & Extraction                                                     |
| IP Geolocation Check        | HTTP Request                               | Enrich data with IP geolocation info   | Extract Booking Data         | AI Agent                          | ## 2. IP Geolocation and AI Analysis                                                         |
| Google Gemini Chat Model    | AI Language Model (Langchain)               | Provides AI model for fraud analysis   | AI Agent (as language model) | AI Agent                          | ## 2. IP Geolocation and AI Analysis                                                         |
| AI Agent                   | Langchain Agent                            | Generate AI fraud risk analysis         | IP Geolocation Check         | Enhanced Risk Calculator           | ## 2. IP Geolocation and AI Analysis                                                         |
| Enhanced Risk Calculator    | Code                                       | Combine AI + rule-based risk scoring   | AI Agent                    | Critical Risk Check, High Risk Check, Log to Google Sheets | ## 3. Risk Calculation and Decision Logic                                                   |
| Critical Risk Check         | If                                         | Check if risk is CRITICAL               | Enhanced Risk Calculator     | Block User Account, Send a message | ## 3. Risk Calculation and Decision Logic                                                   |
| High Risk Check             | If                                         | Check if risk is HIGH                   | Enhanced Risk Calculator     | Flag for Review, Send a message1   | ## 3. Risk Calculation and Decision Logic                                                   |
| Block User Account          | HTTP Request                               | Block user account on critical risk    | Critical Risk Check          | -                                 | ## 4. Action & Notification                                                                 |
| Send a message             | Gmail                                      | Send critical fraud alert email        | Critical Risk Check          | -                                 | ## 4. Action & Notification                                                                 |
| Flag for Review             | HTTP Request                               | Flag transaction for manual review     | High Risk Check             | -                                 | ## 4. Action & Notification                                                                 |
| Send a message1            | Gmail                                      | Send high-risk review notification     | High Risk Check             | -                                 | ## 4. Action & Notification                                                                 |
| Log to Google Sheets        | Google Sheets                              | Log all transaction and risk data      | Enhanced Risk Calculator     | Send Response                     | ## 5. Logging & Response                                                                     |
| Send Response              | Respond to Webhook                         | Return final JSON result to caller     | Log to Google Sheets         | -                                 | ## 5. Logging & Response                                                                     |
| Workflow Overview          | Sticky Note                               | Describes overall workflow              | -                           | -                                 | ## Fraud Detection Workflow Overview                                                        |
| Data Ingestion Notes       | Sticky Note                               | Describes data ingestion block          | -                           | -                                 | ## 1. Initial Data Ingestion & Extraction                                                  |
| Geolocation & AI Notes     | Sticky Note                               | Describes geolocation and AI block      | -                           | -                                 | ## 2. IP Geolocation and AI Analysis                                                      |
| Risk Calculation Notes     | Sticky Note                               | Describes risk calculation block        | -                           | -                                 | ## 3. Risk Calculation and Decision Logic                                              |
| Action & Notification Notes | Sticky Note                               | Describes action and notification block | -                           | -                                 | ## 4. Action & Notification                                                           |
| Logging & Response Notes   | Sticky Note                               | Describes logging and response block    | -                           | -                                 | ## 5. Logging & Response                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Booking Transaction Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `fraud-detection`  
   - Response Mode: `responseNode` (to send response downstream)  

2. **Create Set Node: "Extract Booking Data"**  
   - Extract these values from webhook JSON body:  
     - `user_id` = `{{$json.body.user_id}}`  
     - `booking_amount` = `{{$json.body.amount}}`  
     - `booking_time` = `{{$json.body.timestamp}}`  
     - `ip_address` = `{{$json.body.ip_address}}`  
     - `payment_method` = `{{$json.body.payment_method}}`  
     - `booking_location` = `{{$json.body.location}}`  
     - `session_id` = `{{$json.body.session_id}}`  
   - Connect `Booking Transaction Webhook` → `Extract Booking Data`

3. **Create HTTP Request Node: "IP Geolocation Check"**  
   - Method: GET  
   - URL: `http://ip-api.com/json/{{$node["Extract Booking Data"].json["ip_address"]}}`  
   - Connect `Extract Booking Data` → `IP Geolocation Check`

4. **Create Google Gemini Chat Model Node: "Google Gemini Chat Model"**  
   - Model Name: `models/gemini-2.5-pro`  
   - Configure Google Palm API credentials  
   - No direct input connections; used by AI Agent node.

5. **Create Langchain Agent Node: "AI Agent"**  
   - Text Prompt:  
     ```
     Analyze this booking transaction for potential fraud patterns and suspicious behavior:

     **Transaction Details:**
     - User ID: {{$node['Extract Booking Data'].json['user_id']}}
     - Amount: ${{$node['Extract Booking Data'].json['booking_amount']}}
     - Booking Time: {{$node['Extract Booking Data'].json['booking_time']}}
     - IP Location: {{$node['IP Geolocation Check'].json['city']}}, {{$node['IP Geolocation Check'].json['country']}}
     - Payment Method: {{$node['Extract Booking Data'].json['payment_method']}}
     - Session ID: {{$node['Extract Booking Data'].json['session_id']}}

     **Please provide your analysis in the following JSON format:**
     {
       "risk_score": [number between 0-100],
       "risk_level": "[LOW/MEDIUM/HIGH/CRITICAL]",
       "reasons": ["reason1", "reason2", "reason3"],
       "fraud_indicators": ["indicator1", "indicator2"],
       "recommendation": "[APPROVE/REVIEW/BLOCK]"
     }
     ```
   - System Message:  
     Defines role as advanced fraud detection AI with scoring guidelines (0-25 LOW, 26-50 MEDIUM, 51-75 HIGH, 76-100 CRITICAL).  
   - Connect `IP Geolocation Check` → `AI Agent`  
   - Configure to use the `Google Gemini Chat Model` node as language model.

6. **Create Code Node: "Enhanced Risk Calculator"**  
   - Paste given JavaScript code that:  
     - Parses AI Agent JSON output  
     - Applies rule-based risk scoring (amount thresholds, geolocation failures, time of booking, payment method risks)  
     - Calculates combined risk score and final risk level  
     - Returns detailed object with risk data.  
   - Connect `AI Agent` → `Enhanced Risk Calculator`

7. **Create If Node: "Critical Risk Check"**  
   - Condition: `{{$json.risk_level}} == "CRITICAL"`  
   - Connect `Enhanced Risk Calculator` → `Critical Risk Check`

8. **Create If Node: "High Risk Check"**  
   - Condition: `{{$json.risk_level}} == "HIGH"`  
   - Connect `Enhanced Risk Calculator` → `High Risk Check`

9. **Create HTTP Request Node: "Block User Account"**  
   - Method: POST  
   - URL: `https://oneclicktracker.in/booking/fraud/block-user`  
   - Body Parameters: `user_id` = `{{$node["Extract Booking Data"].json.body.user_id}}`  
   - On Error: continue workflow  
   - Connect `Critical Risk Check` (true) → `Block User Account`

10. **Create Gmail Node: "Send a message" (Critical Alert)**  
    - To: `abc@gmail.com` (replace with actual)  
    - Subject: `🚨 CRITICAL FRAUD ALERT - {{$node["Extract Booking Data"].json.body.user_id}} - ${{$node["Extract Booking Data"].json.body.amount}}`  
    - HTML Body: detailed transaction info, risk factors, fraud indicators, urgent warning  
    - OAuth2 Gmail credentials configured  
    - Connect `Critical Risk Check` (true) → `Send a message`

11. **Create HTTP Request Node: "Flag for Review"**  
    - Method: POST  
    - URL: `https://oneclicktracker.in/booking/fraud/flag-transaction`  
    - Body Parameters: `user_id` = `{{$node["Extract Booking Data"].json.body.user_id}}`  
    - On Error: continue workflow  
    - Connect `High Risk Check` (true) → `Flag for Review`

12. **Create Gmail Node: "Send a message1" (High Risk Alert)**  
    - To: `abc@gmail.com` (replace with actual)  
    - Subject: `⚠️ High Risk Transaction - Review Required - {{$node["Extract Booking Data"].json.body.user_id}}`  
    - HTML Body: summary of risk factors, recommended review actions  
    - OAuth2 Gmail credentials configured  
    - Connect `High Risk Check` (true) → `Send a message1`

13. **Create Google Sheets Node: "Log to Google Sheets"**  
    - Operation: Append  
    - Document ID: Google Sheets document containing fraud logging data  
    - Sheet Name: `gid=0` (Sheet1)  
    - Map columns for user_id, booking_amount, risk_score, risk_level, recommendation, risk_factors, fraud_indicators, ai_analysis, location, ip_address, timestamp, session_id, payment_method, AI raw score, rule-based score  
    - Use service account credentials  
    - Connect `Enhanced Risk Calculator` → `Log to Google Sheets`

14. **Create Respond to Webhook Node: "Send Response"**  
    - Respond with JSON summarizing:  
      - status: "processed"  
      - risk_level, risk_score, recommendation from Enhanced Risk Calculator  
      - message summarizing risk level  
      - actions_taken based on risk level (block, flag, logged)  
      - fraud_indicators_count and session_id  
    - Connect `Log to Google Sheets` → `Send Response`

15. **Connect Branches**  
    - `Critical Risk Check` (false) → `High Risk Check`  
    - Both `Critical Risk Check` and `High Risk Check` false branches → `Log to Google Sheets`

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow leverages the Google Gemini Chat Model (PaLM API) for advanced AI fraud analysis integrated via Langchain. | Requires Google Palm API credentials configured in n8n.                                       |
| Critical and High risk alerts are sent via Gmail nodes using OAuth2 authentication; configure credentials accordingly. | Gmail OAuth2 credentials need to be set up securely in n8n.                                    |
| IP geolocation is done via `http://ip-api.com` free API; consider API rate limits and fallback handling.               | https://ip-api.com/docs/                                                                        |
| Google Sheets node uses service account authentication to log transaction and analysis data for auditing.              | Ensure Google Sheets API is enabled and service account has write access to the target sheet. |
| The AI Agent expects strict JSON output; parsing fallback logic ensures workflow resilience to malformed AI responses. | Parsing errors logged in code node console for troubleshooting.                                |
| Risk scoring thresholds and rules can be customized in the Enhanced Risk Calculator node’s code block as needed.        | Business logic can adapt to evolving fraud patterns.                                          |
| Webhook response includes detailed summary to facilitate upstream system integration and monitoring.                   | Response JSON includes session ID to correlate with original request.                          |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.