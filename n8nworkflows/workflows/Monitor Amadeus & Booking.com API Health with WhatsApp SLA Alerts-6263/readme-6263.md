Monitor Amadeus & Booking.com API Health with WhatsApp SLA Alerts

https://n8nworkflows.xyz/workflows/monitor-amadeus---booking-com-api-health-with-whatsapp-sla-alerts-6263


# Monitor Amadeus & Booking.com API Health with WhatsApp SLA Alerts

### 1. Workflow Overview

This workflow is designed to monitor the health, uptime, and SLA (Service Level Agreement) compliance of two critical travel supplier APIs: Amadeus Flight API and Booking.com Hotel API. It runs automatically every 10 minutes to check the responsiveness and status codes of these APIs. Based on the response, it calculates health metrics, performance ratings, and determines if SLA breaches occur. If any SLA breach is detected, it sends an alert message via WhatsApp; otherwise, it logs normal status messages.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the monitoring process every 10 minutes.
- **1.2 API Requests:** Simultaneously calls both Amadeus Flight API and Booking.com Hotel API to collect health data.
- **1.3 Health & SLA Calculation:** Processes the API responses to compute health status, uptime percentage, SLA compliance, and performance rating.
- **1.4 Alert Decision:** Checks if any SLA breach occurred to decide the next action.
- **1.5 Alert Handling:** Sends WhatsApp messages for SLA breaches or logs normal status messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow on a fixed schedule, ensuring the entire monitoring process runs automatically every 10 minutes.

- **Nodes Involved:**  
  - Monitor Schedule  
  - Sticky Note (Runs every 10 minutes automatically.)

- **Node Details:**

  - **Monitor Schedule**  
    - Type: Schedule Trigger  
    - Configuration: Interval set to every 10 minutes (`minutesInterval: 10`)  
    - Input: None (trigger node)  
    - Output: Triggers downstream API request nodes  
    - Edge Cases: If n8n is down or paused, triggers may be missed; scheduling relies on n8n's internal scheduler.  
    - Sticky Note: "Runs every 10 minutes automatically."

---

#### 2.2 API Requests

- **Overview:**  
  This block performs simultaneous HTTP requests to Amadeus Flight API and Booking.com Hotel API to obtain their current operational status.

- **Nodes Involved:**  
  - Amadeus Flight API  
  - Booking Hotel API  
  - Sticky Note (Tests both the Amadeus Flight API and Booking.com Hotel API simultaneously.)

- **Node Details:**

  - **Amadeus Flight API**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.amadeus.com/v1/reference-data/airlines`  
      - Timeout: 8 seconds  
      - Response Format: JSON, full response enabled  
      - Headers: Sets `User-Agent` to `TravelMonitor/1.0`  
      - Sends headers enabled  
    - Input: Triggered by Monitor Schedule  
    - Output: JSON response including HTTP status code and headers  
    - Edge Cases:  
      - Timeout if API is slow or down  
      - Non-2xx status codes indicate failure or issues  
      - API rate limits or authentication issues (not explicitly handled)  
    - Sticky Note: "Tests both the Amadeus Flight API and Booking.com Hotel API simultaneously."

  - **Booking Hotel API**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://distribution-xml.booking.com/` (likely Booking.com distribution endpoint)  
      - Timeout: 8 seconds  
      - Response Format: JSON, full response enabled  
      - Headers: Sets `User-Agent` to `TravelMonitor/1.0`  
      - Sends headers enabled  
    - Input: Triggered by Monitor Schedule  
    - Output: JSON response including HTTP status code and headers  
    - Edge Cases:  
      - Timeout or slow response  
      - Failures due to authentication or availability issues  
      - Non-2xx status codes are treated as unhealthy  
    - Sticky Note: Same as above.

---

#### 2.3 Health & SLA Calculation

- **Overview:**  
  This code block analyzes API responses, simulates response times, calculates uptime percentages, and evaluates SLA compliance and performance ratings for each supplier.

- **Nodes Involved:**  
  - Calculate Health & SLA  
  - Sticky Note (Processes health, uptime, and SLA data.)

- **Node Details:**

  - **Calculate Health & SLA**  
    - Type: Code (JavaScript)  
    - Configuration: Custom JavaScript code processing all input items (both APIs) simultaneously  
    - Key Logic:  
      - Defines two suppliers: Amadeus Flight API (SLA target 99.5%) and Booking.com Hotel API (SLA target 99.0%)  
      - Checks HTTP status codes (200–299 considered healthy)  
      - Simulates response times: 200ms if headers present; otherwise 5000ms (to simulate poor performance)  
      - Calculates uptime percentage as 100% if healthy, else 0% (simplified model)  
      - Evaluates SLA compliance based on uptime vs SLA target  
      - Sets health status, SLA status, and performance rating emojis  
      - Flags if alert is required for SLA breach  
    - Input: API responses from Amadeus and Booking.com nodes  
    - Output: Array of result objects, each containing detailed SLA and health info for each supplier  
    - Edge Cases:  
      - Simplified uptime calculation might not reflect realistic uptime over time  
      - Response time is simulated, not measured from real timing data  
      - Missing or malformed API responses could cause errors  
    - Sticky Note: "Processes health, uptime, and SLA data."

---

#### 2.4 Alert Decision

- **Overview:**  
  This conditional node routes data based on whether an SLA breach alert is required.

- **Nodes Involved:**  
  - Alert Check  
  - Sticky Note (Routes to appropriate responses based on breach status.)

- **Node Details:**

  - **Alert Check**  
    - Type: If (Boolean condition)  
    - Configuration: Checks if `alert_required` field in JSON data is `true`  
    - Input: Output from Calculate Health & SLA  
    - Output:  
      - True branch: SLA Breach Alert  
      - False branch: Normal Status Log  
    - Edge Cases:  
      - If input data missing or malformed `alert_required` field, condition may fail or misroute  
    - Sticky Note: "Routes to appropriate responses based on breach status."

---

#### 2.5 Alert Handling

- **Overview:**  
  Based on the alert decision, this block either sends WhatsApp messages for breaches or logs normal status messages.

- **Nodes Involved:**  
  - SLA Breach Alert (Debug Helper)  
  - Normal Status Log (Debug Helper)  
  - Send message (WhatsApp)  
  - Sticky Notes:  
    - For SLA Breach Alert and Normal Status Log: "Records results, with alerts for breaches and normal logs for healthy status."  
    - For Send message: "Sends a WhatsApp message for breach alerts."

- **Node Details:**

  - **SLA Breach Alert**  
    - Type: Debug Helper (for logging)  
    - Configuration: Logs the incoming alert breach data  
    - Input: True output of Alert Check  
    - Output: Feeds into Send message node  
    - Edge Cases: None critical; purely logging node  
    - Sticky Note: Included in combined note below

  - **Normal Status Log**  
    - Type: Debug Helper (for logging)  
    - Configuration: Logs normal SLA-compliant status data  
    - Input: False output of Alert Check  
    - Output: Also feeds into Send message node (which appears to send message in both cases)  
    - Edge Cases: None critical; purely logging node  
    - Sticky Note: Included in combined note below

  - **Send message**  
    - Type: WhatsApp  
    - Configuration:  
      - Operation: Send message  
      - Text Body: Uses expression `{{json.logs}}` (though input JSON structure from previous nodes should be verified)  
      - Phone Number ID: Set to `+91999876667877` (likely the sender’s WhatsApp business number)  
      - Recipient Phone Number: `+919886242228` (recipient's phone number)  
      - Credentials: Uses configured WhatsApp API credentials named "WhatsApp-test"  
    - Input: From both SLA Breach Alert and Normal Status Log nodes  
    - Output: None (end of line)  
    - Edge Cases:  
      - WhatsApp API failures due to connectivity, auth, or quota issues  
      - Missing or malformed message payloads may cause send failure  
    - Sticky Note: "Sends a WhatsApp message for breach alerts."  
    - Note: Both breach and normal logs send WhatsApp message here, which may be an error or intentional for logging.

  - **Sticky Note4** (positioned covering SLA Breach Alert and Normal Status Log)  
    - Content: "Records results, with alerts for breaches and normal logs for healthy status."

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                                     | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                         |
|---------------------|-------------------|----------------------------------------------------|-----------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Monitor Schedule     | Schedule Trigger  | Triggers workflow every 10 minutes                  | None                  | Amadeus Flight API, Booking Hotel API | Runs every 10 minutes automatically.                                                              |
| Amadeus Flight API   | HTTP Request      | Checks Amadeus Flight API health                     | Monitor Schedule      | Calculate Health & SLA    | Tests both the Amadeus Flight API and Booking.com Hotel API simultaneously.                        |
| Booking Hotel API    | HTTP Request      | Checks Booking.com Hotel API health                   | Monitor Schedule      | Calculate Health & SLA    | Tests both the Amadeus Flight API and Booking.com Hotel API simultaneously.                        |
| Calculate Health & SLA | Code             | Processes API responses; computes health and SLA status | Amadeus Flight API, Booking Hotel API | Alert Check              | Processes health, uptime, and SLA data.                                                           |
| Alert Check         | If                | Routes flow based on SLA breach alert flag           | Calculate Health & SLA | SLA Breach Alert (true), Normal Status Log (false) | Routes to appropriate responses based on breach status.                                           |
| SLA Breach Alert    | Debug Helper       | Logs SLA breach alerts                                | Alert Check (true)    | Send message             | Records results, with alerts for breaches and normal logs for healthy status.                     |
| Normal Status Log   | Debug Helper       | Logs normal SLA status                                | Alert Check (false)   | Send message             | Records results, with alerts for breaches and normal logs for healthy status.                     |
| Send message        | WhatsApp           | Sends WhatsApp message for alerts or logs            | SLA Breach Alert, Normal Status Log | None                     | Sends a WhatsApp message for breach alerts.                                                       |
| Sticky Note         | Sticky Note        | Notes on Monitor Schedule                             | None                  | None                     | Runs every 10 minutes automatically.                                                              |
| Sticky Note1        | Sticky Note        | Notes on Alert Check routing                          | None                  | None                     | Routes to appropriate responses based on breach status.                                           |
| Sticky Note2        | Sticky Note        | Notes on calculation code                             | None                  | None                     | Processes health, uptime, and SLA data.                                                           |
| Sticky Note3        | Sticky Note        | Notes on WhatsApp message sending                     | None                  | None                     | Sends a WhatsApp message for breach alerts.                                                       |
| Sticky Note4        | Sticky Note        | Notes covering SLA Breach Alert and Normal Status Log | None                  | None                     | Records results, with alerts for breaches and normal logs for healthy status.                     |
| Sticky Note5        | Sticky Note        | Notes on simultaneous API testing                     | None                  | None                     | Tests both the Amadeus Flight API and Booking.com Hotel API simultaneously.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: `Monitor Schedule`  
   - Type: Schedule Trigger  
   - Interval: Every 10 minutes  
   - No input credentials or special config needed.

2. **Create HTTP Request node for Amadeus:**  
   - Name: `Amadeus Flight API`  
   - Type: HTTP Request  
   - URL: `https://api.amadeus.com/v1/reference-data/airlines`  
   - Method: GET (default)  
   - Timeout: 8000 ms  
   - Response Format: JSON, full response enabled  
   - Send Headers: Enabled  
   - Add Header: `User-Agent` = `TravelMonitor/1.0`  
   - Connect input to `Monitor Schedule`.

3. **Create HTTP Request node for Booking.com:**  
   - Name: `Booking Hotel API`  
   - Type: HTTP Request  
   - URL: `https://distribution-xml.booking.com/`  
   - Method: GET (default)  
   - Timeout: 8000 ms  
   - Response Format: JSON, full response enabled  
   - Send Headers: Enabled  
   - Add Header: `User-Agent` = `TravelMonitor/1.0`  
   - Connect input to `Monitor Schedule`.

4. **Create a Code node to calculate health and SLA:**  
   - Name: `Calculate Health & SLA`  
   - Type: Code (JavaScript)  
   - Use the supplied JavaScript logic:  
     - Process both API responses  
     - Check HTTP status codes for health  
     - Calculate uptime, SLA compliance, performance rating  
     - Return structured JSON results  
   - Connect inputs from both `Amadeus Flight API` and `Booking Hotel API`.

5. **Create an If node to check for alerts:**  
   - Name: `Alert Check`  
   - Type: If  
   - Condition: Check if `alert_required` field is `true`  
   - Connect input from `Calculate Health & SLA`.

6. **Create Debug Helper node for SLA breach alert logging:**  
   - Name: `SLA Breach Alert`  
   - Type: Debug Helper  
   - Connect true output of `Alert Check` here.

7. **Create Debug Helper node for normal status logging:**  
   - Name: `Normal Status Log`  
   - Type: Debug Helper  
   - Connect false output of `Alert Check` here.

8. **Create WhatsApp node to send messages:**  
   - Name: `Send message`  
   - Type: WhatsApp  
   - Operation: Send message  
   - Text Body: `{{json.logs}}` or adapt as needed to reflect alert or log messages  
   - Phone Number ID: Set to your WhatsApp business phone number ID (example: `+91999876667877`)  
   - Recipient Phone Number: Set to intended receiver's number (example: `+919886242228`)  
   - Credentials: Configure WhatsApp API credentials with proper authentication  
   - Connect input from both `SLA Breach Alert` and `Normal Status Log`.

9. **Add Sticky Notes at appropriate places for documentation:**  
   - Near `Monitor Schedule`: "Runs every 10 minutes automatically."  
   - Near API Request nodes: "Tests both the Amadeus Flight API and Booking.com Hotel API simultaneously."  
   - Near `Calculate Health & SLA`: "Processes health, uptime, and SLA data."  
   - Near `Alert Check`: "Routes to appropriate responses based on breach status."  
   - Near `SLA Breach Alert` and `Normal Status Log`: "Records results, with alerts for breaches and normal logs for healthy status."  
   - Near `Send message`: "Sends a WhatsApp message for breach alerts."

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                            |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow uses a simplified uptime calculation model for demonstration purposes, not real historical uptime tracking. | Internal logic note                                        |
| WhatsApp node requires valid WhatsApp Business API credentials and proper phone number configuration. | n8n credential setup documentation                         |
| API URLs and endpoints should be verified for authentication and access permissions before use. | Amadeus and Booking.com API documentation                   |
| Consider adding real response time measurement for improved performance rating accuracy.         | Performance monitoring best practices                      |
| Slack, email, or other notification channels can be integrated similarly by adding relevant nodes. | Expansion ideas                                            |

---

**Disclaimer:** The provided content is exclusively derived from an n8n workflow automation. The processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.