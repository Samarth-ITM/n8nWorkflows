Monitor Elderly Health Vitals & Send Alerts with Apple Health, Twilio & Gmail

https://n8nworkflows.xyz/workflows/monitor-elderly-health-vitals---send-alerts-with-apple-health--twilio---gmail-4563


# Monitor Elderly Health Vitals & Send Alerts with Apple Health, Twilio & Gmail

### 1. Workflow Overview

This workflow, named **Elderwatch**, is designed to monitor elderly health vitals received via an Apple Health integration or similar device through a webhook. It processes key health metrics—heart rate, blood oxygen level, and walking symmetry—to determine if any readings fall outside safe ranges. Based on this analysis, it conditionally triggers alert mechanisms:

- **1.1 Input Reception:** Receives health data via a POST webhook from a mobile device or health monitoring tool.
- **1.2 Health Data Processing:** Parses and evaluates the vitals, flags issues if values are abnormal.
- **1.3 Decision Making:** Determines if immediate attention is required.
- **1.4 Alerting:** If attention is needed, initiates a Twilio voice call and sends a warning email.
- **1.5 Normal Reporting:** If all vitals are normal, sends an all-clear email to caretakers.

Logical blocks correspond to these functions, with clear node chaining from input to conditional branching and notification dispatch.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming POST requests containing health vital data from a phone or health device.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: `Webhook` (Trigger)  
    - Role: Receives POST requests at path `/elderwatch`  
    - Configuration: HTTP method POST, default options  
    - Input: External HTTP POST request JSON payload with health data: `heartrate`, `bloodoxygen`, `walksymmetry`  
    - Output: Passes JSON data downstream  
    - Edge cases: Missing or malformed JSON input, unexpected data types, webhook authentication (not configured here)  
    - Version: 2  

  - **Sticky Note** (positioned nearby)  
    - Content: "Webhook called from phone. Processor determines if health scores need immediate attention."  
    - Role: Documentation only, no technical function

---

#### 2.2 Health Data Processing & Flagging

- **Overview:**  
  Parses incoming vital signs, applies safety thresholds, and compiles a summary report with flags for abnormalities.

- **Nodes Involved:**  
  - Process & Flag Health

- **Node Details:**

  - **Process & Flag Health**  
    - Type: `Code` (JavaScript)  
    - Role: Extracts vitals, validates numeric input, checks if values are out of safe ranges (heart rate < 50 or > 100 bpm, blood oxygen < 92%, walking symmetry < 0.9), flags issues, and constructs a summary string.  
    - Configuration:  
      - Uses a helper function `safeParse` to convert inputs safely to floats with fallback zero.  
      - Threshold checks set boolean `alert` flag and accumulate issue descriptions.  
      - Outputs JSON including `summary` string, boolean `attentionRequired`, issue list, and numeric vitals.  
    - Input: JSON from Webhook node  
    - Output: JSON with processed results and flags  
    - Key Expressions: None external; all logic inside JS code  
    - Edge cases: Invalid or missing input fields, non-numeric values, zero fallback could mask missing data, potential for logic errors in condition checks  
    - Version: 2  

---

#### 2.3 Decision Making: Attention Check

- **Overview:**  
  Branches workflow based on whether any vital sign is flagged as requiring attention.

- **Nodes Involved:**  
  - Attention Required?

- **Node Details:**

  - **Attention Required?**  
    - Type: `If` node  
    - Role: Evaluates if `attentionRequired` boolean is true in processed data  
    - Configuration: Condition checks if `{{$json.attentionRequired}}` is exactly `true` (strict boolean)  
    - Input: From "Process & Flag Health" node output  
    - Output: Two branches:  
      - True branch: leads to alerting nodes  
      - False branch: leads to all-clear email node  
    - Edge cases: Data type mismatch (e.g., string "true"), missing field causing false negatives  
    - Version: 2.2  

---

#### 2.4 Alerting: Twilio Call and Warning Email

- **Overview:**  
  If attention is required, triggers a Twilio call to caretakers with voice alert and sends an email warning.

- **Nodes Involved:**  
  - Twilio Call  
  - Warning Email  
  - Sticky Note1

- **Node Details:**

  - **Twilio Call**  
    - Type: `HTTP Request`  
    - Role: Sends POST request to Twilio Calls API to initiate an outbound phone call  
    - Configuration:  
      - URL: `https://api.twilio.com/2010-04-01/Accounts/<accountid>/Calls.json` (account id placeholder)  
      - Method: POST  
      - Authentication: HTTP Basic with Twilio credentials  
      - Body parameters: From number, To number, and a URL for Twilio to fetch call instructions (`https://n8n.domain.com/webhook/twilio-call`)  
      - Query parameters include `lead` phone and health summary text  
      - Headers set content-type `application/x-www-form-urlencoded`  
    - Input: Condition True from "Attention Required?" node, uses data from "Process & Flag Health" summary  
    - Output: Triggers "Warning Email" on success  
    - Edge cases: Authentication failure, network errors, invalid phone numbers, Twilio API errors, missing credentials  
    - Version: 4.2  

  - **Warning Email**  
    - Type: `Gmail`  
    - Role: Sends an alert email to caretaker with detailed health summary and warning message  
    - Configuration:  
      - Recipient: `xyz@gmail.com` (example)  
      - Subject: Includes alert icon and current date  
      - Message body: HTML formatted with summary and conditional attention text  
      - Credentials: Gmail OAuth2 configured  
    - Input: Triggered by successful Twilio call node  
    - Output: None (end of alert branch)  
    - Edge cases: OAuth token expiry, email sending limits, invalid recipient email  
    - Version: 2.1  

  - **Sticky Note1**  
    - Content: "Twilio HTTP Calls API to make a call and also email."  
    - Role: Documentation  

---

#### 2.5 Normal Reporting: All Clear Email

- **Overview:**  
  If no urgent issues are detected, sends a routine health summary email confirming all vitals are normal.

- **Nodes Involved:**  
  - All Clear Gmail  
  - Sticky Note2

- **Node Details:**

  - **All Clear Gmail**  
    - Type: `Gmail`  
    - Role: Sends routine health update email indicating all vitals are within normal ranges  
    - Configuration:  
      - Recipient: `xyz@gmail.com` (example)  
      - Subject: Includes all-clear icon and current date  
      - Message body: Similar HTML template to warning email but with positive messaging  
      - Credentials: Gmail OAuth2 configured (distinct credential ID from Warning Email)  
    - Input: False branch of "Attention Required?" node  
    - Output: None (end of normal branch)  
    - Edge cases: Same as Warning Email node  
    - Version: 2.1  

  - **Sticky Note2**  
    - Content: "Health vitals are ok -email to caretaker."  
    - Role: Documentation  

---

### 3. Summary Table

| Node Name           | Node Type      | Functional Role                           | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                         |
|---------------------|----------------|-----------------------------------------|-----------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Webhook             | Webhook        | Receives health vitals POST request     | (none)                | Process & Flag Health    | "Webhook called from phone. Processor determines if health scores need immediate attention."      |
| Process & Flag Health| Code           | Parses and flags abnormal vitals        | Webhook               | Attention Required?      |                                                                                                   |
| Attention Required?  | If             | Branches on attentionRequired flag      | Process & Flag Health  | Twilio Call, All Clear Gmail |                                                                                                   |
| Twilio Call         | HTTP Request   | Initiates outgoing alert call via Twilio| Attention Required? (True) | Warning Email          | "Twilio HTTP Calls API to make a call and also email."                                            |
| Warning Email       | Gmail          | Sends alert email with health summary   | Twilio Call           | (none)                   |                                                                                                   |
| All Clear Gmail     | Gmail          | Sends all-clear routine health email    | Attention Required? (False) | (none)                | "Health vitals are ok -email to caretaker."                                                       |
| Sticky Note         | Sticky Note    | Documentation                           | (none)                | (none)                   | "Webhook called from phone. Processor determines if health scores need immediate attention."      |
| Sticky Note1        | Sticky Note    | Documentation                           | (none)                | (none)                   | "Twilio HTTP Calls API to make a call and also email."                                            |
| Sticky Note2        | Sticky Note    | Documentation                           | (none)                | (none)                   | "Health vitals are ok -email to caretaker."                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Webhook`  
   - Type: Webhook  
   - HTTP Method: `POST`  
   - Path: `elderwatch`  
   - Purpose: Receive health vitals JSON payload with fields `heartrate`, `bloodoxygen`, `walksymmetry`.

2. **Add a Code node**  
   - Name: `Process & Flag Health`  
   - Type: Code (JavaScript)  
   - Connect input from `Webhook`  
   - Paste the following logic:  

     ```javascript
     function safeParse(value, fallback = 0) {
       const num = parseFloat(value);
       return isNaN(num) ? fallback : num;
     }

     const heartRate = safeParse($input.first().json.body.heartrate, 0);
     const bloodOxygen = safeParse($input.first().json.body.bloodoxygen, 0);
     const walkSymmetry = safeParse($input.first().json.body.walksymmetry, 0);

     let alert = false;
     let issues = [];

     if (heartRate > 100 || heartRate < 50) {
       alert = true;
       issues.push(`❤️ Heart rate out of range: ${heartRate.toFixed(0)} bpm`);
     }
     if (bloodOxygen < 92) {
       alert = true;
       issues.push(`🩸 Blood oxygen low: ${bloodOxygen.toFixed(0)}%`);
     }
     if (walkSymmetry < 0.9) {
       alert = true;
       issues.push(`🚶 Walking symmetry off: ${walkSymmetry}`);
     }

     const summary = `📋 Daily Health Report

     ❤️ Heart Rate: ${heartRate.toFixed(0)} bpm
     🩸 Blood Oxygen: ${bloodOxygen.toFixed(0)}%
     🚶 Walking Symmetry: ${walkSymmetry >= 0.9 ? "Good" : "Needs Attention"}

     ${alert ? `⚠️ Attention Required:\n${issues.join("\n")}` : "✅ All vitals within range."}
     `;

     return [
       {
         json: {
           summary,
           attentionRequired: alert,
           issues,
           heartRate,
           bloodOxygen,
           walkSymmetry
         }
       }
     ];
     ```

3. **Add an If node**  
   - Name: `Attention Required?`  
   - Type: If  
   - Connect input from `Process & Flag Health`  
   - Condition: Check if `{{$json.attentionRequired}}` is `true` (strict boolean)  
   - Configure two outputs: True and False

4. **Add an HTTP Request node for Twilio call**  
   - Name: `Twilio Call`  
   - Type: HTTP Request  
   - Connect True output of `Attention Required?` to this node  
   - Method: POST  
   - URL: `https://api.twilio.com/2010-04-01/Accounts/<accountid>/Calls.json` (replace `<accountid>` with actual Twilio Account SID)  
   - Authentication: HTTP Basic with Twilio credentials (username = Account SID, password = Auth Token)  
   - Body Parameters (form-urlencoded):  
     - From: Your verified Twilio number (e.g., `+44700000000`)  
     - To: Recipient phone number (e.g., `+44712121211`)  
     - Url: `https://n8n.domain.com/webhook/twilio-call` (this webhook URL should host TwiML instructions for the call)  
   - Query Parameters: Optional lead phone and `summary` from previous node (expression: `{{ $('Process & Flag Health').item.json.summary }}`)  
   - Headers: `content-type: application/x-www-form-urlencoded`  
   - Connect output to next node `Warning Email`

5. **Add Gmail node to send warning email**  
   - Name: `Warning Email`  
   - Type: Gmail  
   - Connect input from `Twilio Call`  
   - Configure credentials with Gmail OAuth2 account  
   - Recipient: `xyz@gmail.com` (replace with actual caretaker email)  
   - Subject: `⚠️ Alert: Attention Needed – Vitals Out of Range for {{ $now.format('yyyy-MM-dd') }}`  
   - Message (HTML): Include an `<h2>`, summary text (use `{{ $json.summary }}`), conditional message if attention is required, and footer note  
   - No output node (end of alert branch)

6. **Add Gmail node to send all-clear email**  
   - Name: `All Clear Gmail`  
   - Type: Gmail  
   - Connect False output of `Attention Required?`  
   - Configure credentials with Gmail OAuth2 account (can be same or different)  
   - Recipient: `xyz@gmail.com`  
   - Subject: `✅ All Clear: Health Summary for {{ $now.format('yyyy-MM-dd') }}`  
   - Message: Similar HTML template to Warning Email, but positive messaging indicating all vitals are normal  
   - No output node (end of normal branch)

7. **Add Sticky Notes** (optional but recommended for clarity)  
   - Near `Webhook`: Note "Webhook called from phone. Processor determines if health scores need immediate attention."  
   - Near `Twilio Call` and `Warning Email`: Note "Twilio HTTP Calls API to make a call and also email."  
   - Near `All Clear Gmail`: Note "Health vitals are ok -email to caretaker."

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses Twilio Call API to trigger a voice alert followed by an email notification to caretakers when abnormal vitals are detected.| Twilio API documentation: https://www.twilio.com/docs/voice/api/call-resource                       |
| Gmail nodes require OAuth2 credentials with appropriate scopes for sending emails.                                                         | Gmail OAuth2 setup guide: https://developers.google.com/identity/protocols/oauth2                    |
| The Twilio call URL (`Url` parameter) must point to a valid TwiML webhook that instructs Twilio what to say during the call.                | Example TwiML: https://www.twilio.com/docs/voice/twiml                                                |
| Thresholds used are general guidelines and should be customized based on medical advice.                                                    | Health vitals reference: https://www.healthline.com/health/normal-heart-rate                        |
| The workflow assumes JSON payload keys: `heartrate`, `bloodoxygen`, and `walksymmetry` in the webhook POST body.                          | Payload format must be consistent with Apple Health export or the source device                     |

---

This completes the detailed documentation and reconstruction guide for the **Elderwatch** n8n workflow.