Track Flight Fares with Amadeus & Skyscanner - Alerts, Refunds & Trends

https://n8nworkflows.xyz/workflows/track-flight-fares-with-amadeus---skyscanner---alerts--refunds---trends-6233


# Track Flight Fares with Amadeus & Skyscanner - Alerts, Refunds & Trends

### 1. Workflow Overview

This workflow automates post-booking flight fare tracking to detect fare drops and trigger alerts and refund checks using Amadeus and Skyscanner APIs. It targets users who want to monitor already booked flights for potential savings, receive notifications, and initiate refund processes if applicable. The workflow is structured into distinct logical blocks:

- **1.1 Scheduled Trigger & Data Retrieval**: Periodically fetch tracked bookings requiring fare updates.
- **1.2 Fare Search Preparation & API Queries**: Prepare search parameters and query Amadeus and Skyscanner for current fare offers.
- **1.3 Fare Analysis & Decision Logic**: Analyze retrieved fares, detect savings, evaluate trends, and assess refund eligibility.
- **1.4 Data Persistence & Status Update**: Update fare tracking records and booking statuses in the database.
- **1.5 Notification Dispatch**: Conditionally send fare drop notifications via email, SMS, and Slack.
- **1.6 Refund Automation**: If eligible, initiate refund requests through an airline booking system API.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

**Overview:**  
This block initiates the workflow every 6 hours and retrieves up to 50 tracked bookings that are confirmed, upcoming within 1 to 90 days, and require fare checking.

**Nodes Involved:**  
- Fare Check Trigger  
- Get Tracked Bookings  

**Node Details:**  

- **Fare Check Trigger**  
  - Type: Schedule Trigger  
  - Role: Entry point to periodically start the workflow every 6 hours.  
  - Config: Interval set to 6 hours.  
  - Inputs: None  
  - Outputs: Triggers "Get Tracked Bookings"  
  - Edge cases: Trigger misfires, schedule conflicts.

- **Get Tracked Bookings**  
  - Type: Postgres (Execute Query)  
  - Role: Fetches bookings with tracking enabled, confirmed status, and needing fare updates.  
  - Config: SQL query selects booking fields, joins with fare_tracking for current data, filters bookings departing between 1 and 90 days from now, limits to 50.  
  - Inputs: Trigger node  
  - Outputs: Booking list to "Prepare Fare Search"  
  - Credentials: Postgres database access  
  - Edge cases: DB connection errors, empty result set, query timeouts.

---

#### 2.2 Fare Search Preparation & API Queries

**Overview:**  
Prepares the booking data into standardized fare search parameters and queries Amadeus and Skyscanner APIs concurrently for current flight offers.

**Nodes Involved:**  
- Prepare Fare Search  
- Search Current Fares (Amadeus API)  
- Search Skyscanner Fares (Skyscanner API)  

**Node Details:**  

- **Prepare Fare Search**  
  - Type: Code  
  - Role: Formats data for API requests, extracts and converts fields such as dates and fares.  
  - Config: JavaScript code loops over bookings, formats departure date to ISO string, creates objects with relevant search parameters.  
  - Inputs: Bookings list from DB query  
  - Outputs: Structured search parameters to both API request nodes  
  - Edge cases: Date parsing errors, missing fields.

- **Search Current Fares**  
  - Type: HTTP Request  
  - Role: Queries Amadeus flight offers API with batched requests (batch size 5, interval 2s).  
  - Config: Uses Bearer token from Amadeus credentials, sends GET requests to Amadeus endpoint with parameters derived from input JSON.  
  - Inputs: Prepared search params  
  - Outputs: API response JSON to "Analyze Fare Drops"  
  - Credentials: Amadeus API token  
  - Edge cases: Auth failures, rate limits, API downtime, malformed responses.

- **Search Skyscanner Fares**  
  - Type: HTTP Request  
  - Role: Queries Skyscanner browse API with batched requests (batch size 3, interval 3s).  
  - Config: Uses API key header from credentials, constructs URL dynamically using origin, destination, and departure date.  
  - Inputs: Prepared search params  
  - Outputs: API response JSON to "Analyze Fare Drops"  
  - Credentials: Skyscanner API key  
  - Edge cases: Auth failures, rate limits, invalid parameters, API errors.

---

#### 2.3 Fare Analysis & Decision Logic

**Overview:**  
This critical block analyzes fare data from both APIs, compares to original and current lowest fares, calculates savings and trends, determines notification priority, and assesses refund eligibility based on airline-specific policies.

**Nodes Involved:**  
- Analyze Fare Drops  

**Node Details:**  

- **Analyze Fare Drops**  
  - Type: Code  
  - Role: Processes Amadeus and Skyscanner fare offers, finds new lowest fare, computes savings and percentage, sets fare trend (dropping, rising, stable), priorities, and refund eligibility.  
  - Config:  
    - Uses JavaScript to parse multiple API responses  
    - Checks fare values and segments for each offer  
    - Refund eligibility based on airline code, savings amount, and booking age with a simplified policy map  
    - Generates output JSON with enriched fare tracking data, including available fares (top 5 sorted), timestamp, and days until departure.  
  - Inputs: Output of both fare search API nodes and prepared search data  
  - Outputs: Fare analysis result to "Update Fare Tracking"  
  - Edge cases: Missing or malformed API data, unexpected formats, division by zero in percentage calculations, policy misconfigurations.

---

#### 2.4 Data Persistence & Status Update

**Overview:**  
Stores the updated fare tracking information and updates the booking status with the new lowest fare and last check timestamp.

**Nodes Involved:**  
- Update Fare Tracking  
- Update Booking Status  

**Node Details:**  

- **Update Fare Tracking**  
  - Type: Postgres (Execute Query)  
  - Role: Inserts or updates fare tracking record with latest fare info, trend, priority, refund eligibility, and available fares JSON.  
  - Config: SQL uses UPSERT pattern ("ON CONFLICT") keyed by booking_id.  
  - Inputs: Fare analysis JSON  
  - Outputs: Data to "Update Booking Status"  
  - Credentials: Postgres DB  
  - Edge cases: DB transaction failures, JSON serialization errors for available fares.

- **Update Booking Status**  
  - Type: Postgres (Execute Query)  
  - Role: Updates booking record with last_checked timestamp and current_lowest_fare value.  
  - Config: SQL update query using booking_id and new lowest fare.  
  - Inputs: Fare tracking update output  
  - Outputs: Trigger to "Check if Notification Needed"  
  - Credentials: Postgres DB  
  - Edge cases: DB errors, stale data updates.

---

#### 2.5 Notification Dispatch

**Overview:**  
Decides if notifications are needed based on fare drop significance and priority, then sends email, Slack, and optionally SMS alerts to passengers and internal teams.

**Nodes Involved:**  
- Check if Notification Needed  
- Send Fare Drop Email  
- Check if SMS Needed  
- Send SMS Alert  
- Notify Slack Team  

**Node Details:**  

- **Check if Notification Needed**  
  - Type: If  
  - Role: Filters for bookings where action_recommended is true (significant savings).  
  - Inputs: Booking update output  
  - Outputs: Triggers downstream notification nodes conditionally.  
  - Edge cases: Expression evaluation failures.

- **Send Fare Drop Email**  
  - Type: HTTP Request  
  - Role: Sends personalized email alerts via SendGrid with dynamic template data including savings, flight details, refund eligibility.  
  - Config: Uses SendGrid API key, templated JSON body with personalization fields.  
  - Inputs: Conditional pass from "Check if Notification Needed"  
  - Credentials: SendGrid API key  
  - Edge cases: API rate limits, invalid email addresses, template errors.

- **Check if SMS Needed**  
  - Type: If  
  - Role: Filters only high-priority notifications to send SMS alerts via Twilio.  
  - Inputs: Output of "Check if Notification Needed"  
  - Outputs: Triggers "Send SMS Alert" if priority equals "high".  
  - Edge cases: Logic errors, missing priority field.

- **Send SMS Alert**  
  - Type: HTTP Request  
  - Role: Sends SMS messages via Twilio with fare drop summary and refund eligibility note.  
  - Config: Uses Twilio account SID and phone number credentials, basic auth.  
  - Inputs: High priority filtered data  
  - Credentials: Twilio credentials  
  - Edge cases: SMS sending failure, invalid phone numbers, auth errors.

- **Notify Slack Team**  
  - Type: HTTP Request  
  - Role: Posts a formatted message to a Slack channel with fare drop summary for internal monitoring.  
  - Config: Sends POST request with Bearer token auth, message includes flight info, savings, refund status, priority.  
  - Inputs: Always triggered for notified bookings  
  - Credentials: Slack API token  
  - Edge cases: Slack API failures, invalid channels, auth errors.

---

#### 2.6 Refund Automation

**Overview:**  
For bookings eligible for refunds, automatically initiates refund requests via an airline booking system API.

**Nodes Involved:**  
- Check Refund Eligible  
- Initiate Refund Process  

**Node Details:**  

- **Check Refund Eligible**  
  - Type: If  
  - Role: Checks refund_eligible boolean from fare analysis result.  
  - Inputs: Output of "Notify Slack Team" (final notification branch)  
  - Outputs: Triggers refund initiation if true.  
  - Edge cases: False positives/negatives due to policy logic.

- **Initiate Refund Process**  
  - Type: HTTP Request  
  - Role: Sends refund initiation request with booking confirmation, passenger email, fare details, and reason "fare_drop_detected".  
  - Config: Uses airline system API token in headers, POST request with JSON body.  
  - Inputs: Refund-eligible filtered data  
  - Credentials: Airline booking system API token  
  - Edge cases: API failures, incorrect data, refund denial, authorization errors.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                                | Input Node(s)             | Output Node(s)                                    | Sticky Note                                                                                                  |
|-------------------------|----------------------|------------------------------------------------|---------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Fare Check Trigger       | Schedule Trigger     | Starts workflow every 6 hours                   | None                      | Get Tracked Bookings                             | Fundamental Aspects: Initiates the workflow                                                                 |
| Get Tracked Bookings     | Postgres             | Retrieves bookings needing fare tracking        | Fare Check Trigger        | Prepare Fare Search                              | Fundamental Aspects: Retrieves existing booking data                                                        |
| Prepare Fare Search      | Code                 | Prepares search parameters for APIs             | Get Tracked Bookings      | Search Current Fares, Search Skyscanner Fares   | Fundamental Aspects: Prepares query parameters                                                              |
| Search Current Fares     | HTTP Request         | Queries Amadeus API for current fares            | Prepare Fare Search       | Analyze Fare Drops                               | Fundamental Aspects: Queries Skyscanner for current fares                                                   |
| Search Skyscanner Fares  | HTTP Request         | Queries Skyscanner API for current fares         | Prepare Fare Search       | Analyze Fare Drops                               | Fundamental Aspects: Queries Skyscanner for current fares                                                   |
| Analyze Fare Drops       | Code                 | Analyzes fares, savings, trends, refund eligibility | Search Current Fares, Search Skyscanner Fares, Prepare Fare Search | Update Fare Tracking                         | Fundamental Aspects: Identifies significant fare reductions                                                 |
| Update Fare Tracking     | Postgres             | Inserts/updates fare tracking data                | Analyze Fare Drops        | Update Booking Status                            | Fundamental Aspects: Updates fare tracking records                                                          |
| Update Booking Status    | Postgres             | Updates booking last checked timestamp and fare  | Update Fare Tracking      | Check if Notification Needed                     | Fundamental Aspects: Updates status based on fare changes                                                   |
| Check if Notification Needed | If               | Determines if notifications should be sent       | Update Booking Status     | Send Fare Drop Email, Check if SMS Needed, Notify Slack Team | Fundamental Aspects: Determines if alerts are required                                                      |
| Send Fare Drop Email     | HTTP Request         | Sends fare drop notification email via SendGrid | Check if Notification Needed | None                                           | Fundamental Aspects: Notifies users via email                                                              |
| Check if SMS Needed      | If                   | Checks if SMS alert is necessary (only high priority) | Check if Notification Needed | Send SMS Alert                                  | Fundamental Aspects: Decides if SMS alert is necessary                                                     |
| Send SMS Alert           | HTTP Request         | Sends SMS alerts via Twilio                       | Check if SMS Needed       | None                                            | Fundamental Aspects: Sends SMS notification                                                                |
| Notify Slack Team        | HTTP Request         | Posts fare drop notification to Slack channel    | Check if Notification Needed | Check Refund Eligible                            | Fundamental Aspects: Alerts the team via Slack                                                             |
| Check Refund Eligible    | If                   | Checks if refund process should be initiated      | Notify Slack Team         | Initiate Refund Process                          | Fundamental Aspects: Assesses refund eligibility                                                           |
| Initiate Refund Process  | HTTP Request         | Initiates refund via airline booking API          | Check Refund Eligible     | None                                            | Fundamental Aspects: Starts refund procedure if eligible                                                   |
| Sticky Note              | Sticky Note          | Describes fundamental workflow aspects            | None                      | None                                            | Fundamental Aspects: Summary of workflow blocks and nodes                                                  |
| Sticky Note1             | Sticky Note          | Lists required external resources                  | None                      | None                                            | Required Resources: API credentials, service integrations, internet access                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: "Fare Check Trigger"  
   - Set interval to every 6 hours.

2. **Create Postgres Node**  
   - Name: "Get Tracked Bookings"  
   - Configure credentials for your Postgres DB.  
   - Use SQL query to select bookings with tracking enabled, confirmed, departing between 1 and 90 days, and not checked in last 6 hours. Limit to 50 rows.

3. **Create Code Node**  
   - Name: "Prepare Fare Search"  
   - JavaScript to map each booking: format departure date as ISO (yyyy-mm-dd), parse fares as floats, include passenger and booking details, add current timestamp for last_checked.  
   - Input: Output from "Get Tracked Bookings".

4. **Create HTTP Request Node**  
   - Name: "Search Current Fares"  
   - Method: GET  
   - URL: `https://api.amadeus.com/v2/shopping/flight-offers` (parameters to be set dynamically from input)  
   - Authentication: HTTP Header Bearer token with Amadeus credentials  
   - Enable batching: batch size 5, interval 2 seconds.

5. **Create HTTP Request Node**  
   - Name: "Search Skyscanner Fares"  
   - Method: GET  
   - URL template: `https://api.skyscanner.com/browse/v1.0/US/USD/en-US/{{ $json.origin }}/{{ $json.destination }}/{{ $json.departure_date }}`  
   - Authentication: HTTP Header with `x-api-key` from Skyscanner credentials  
   - Enable batching: batch size 3, interval 3 seconds.

6. **Connect "Prepare Fare Search" output to both "Search Current Fares" and "Search Skyscanner Fares".**

7. **Create Code Node**  
   - Name: "Analyze Fare Drops"  
   - Implement JavaScript that:  
     - Reads fares from both APIs and original/current fares.  
     - Identifies new lowest fare and source.  
     - Calculates savings amounts and percentages.  
     - Determines fare trend (dropping, stable, rising).  
     - Sets priority (high, medium, low) based on savings thresholds.  
     - Checks refund eligibility using predefined airline policies and booking age.  
     - Outputs enriched fare data and metadata (timestamp, days to departure, available fares list).  
   - Inputs: Output from both API fare search nodes and "Prepare Fare Search".

8. **Create Postgres Node**  
   - Name: "Update Fare Tracking"  
   - Configure to insert or update fare tracking table with all calculated data using UPSERT pattern keyed on booking_id.  
   - Use parameters for all fare tracking fields (lowest fare, savings, trend, priority, refund eligibility, available fares JSON).  
   - Credentials: Postgres DB.

9. **Create Postgres Node**  
   - Name: "Update Booking Status"  
   - Update bookings table with last_checked timestamp and current_lowest_fare.  
   - Credentials: Postgres DB.

10. **Create If Node**  
    - Name: "Check if Notification Needed"  
    - Condition: Check if `action_recommended` is true.

11. **Create HTTP Request Node**  
    - Name: "Send Fare Drop Email"  
    - Configure SendGrid API with API key in headers.  
    - POST to `https://api.sendgrid.com/v3/mail/send` with JSON body using dynamic template data for passenger info, fares, savings, refund status, priority.  
    - Input: true branch from "Check if Notification Needed".

12. **Create If Node**  
    - Name: "Check if SMS Needed"  
    - Condition: `priority` equals "high".

13. **Create HTTP Request Node**  
    - Name: "Send SMS Alert"  
    - Configure Twilio HTTP Basic Auth with account SID and auth token.  
    - POST to Twilio Messages endpoint with To, From, and Body containing fare drop summary.  
    - Input: true branch from "Check if SMS Needed".

14. **Create HTTP Request Node**  
    - Name: "Notify Slack Team"  
    - POST to Slack chat.postMessage API with Bearer token authorization.  
    - Message includes flight and savings details, refund eligibility, and priority.  
    - Input: true branch from "Check if Notification Needed".

15. **Create If Node**  
    - Name: "Check Refund Eligible"  
    - Condition: `refund_eligible` equals true.

16. **Create HTTP Request Node**  
    - Name: "Initiate Refund Process"  
    - POST to airline booking system refund API endpoint with confirmation code, passenger email, fare details, and reason "fare_drop_detected".  
    - Use Bearer token auth with airline system API credentials.  
    - Input: true branch from "Check Refund Eligible".

17. **Wire nodes in the order described in the connections section:**  
    - Schedule Trigger → Get Tracked Bookings → Prepare Fare Search → (Search Current Fares + Search Skyscanner Fares) → Analyze Fare Drops → Update Fare Tracking → Update Booking Status → Check if Notification Needed → (Send Fare Drop Email, Check if SMS Needed → Send SMS Alert, Notify Slack Team) → Check Refund Eligible → Initiate Refund Process.

18. **Add Sticky Notes** for documentation and resource reminders as per original layout.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow requires valid API credentials for Amadeus, Skyscanner, SendGrid, Twilio, Slack, and airline system API. | API credential management and secure storage are critical.                                     |
| Slack API URL used is `https://api.slack.com/api/chat.postMessage` (see Slack API docs for details). | Official Slack API documentation: https://api.slack.com/methods/chat.postMessage               |
| SendGrid email template ID used: `d-fare-drop-alert` — customize this template in SendGrid dashboard. | SendGrid templates documentation: https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-transactional-templates |
| Twilio SMS requires account SID, auth token, and a verified phone number for sending.              | Twilio docs: https://www.twilio.com/docs/sms                                                       |
| Airlines refund policies in the code are simplified and may need adjustments to real policies.     | Airline-specific refund eligibility logic is customizable in the "Analyze Fare Drops" node.    |
| Workflow designed to handle up to 50 bookings per run to avoid API rate limits and DB load.        | Consider increasing batch sizes or adding pagination for larger deployments.                    |
| Ensure n8n instance has internet access and proper network permissions to reach all external APIs. | Network and credential setup essential for external API calls.                                 |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.