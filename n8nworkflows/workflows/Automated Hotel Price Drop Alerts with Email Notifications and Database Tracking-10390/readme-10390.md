Automated Hotel Price Drop Alerts with Email Notifications and Database Tracking

https://n8nworkflows.xyz/workflows/automated-hotel-price-drop-alerts-with-email-notifications-and-database-tracking-10390


# Automated Hotel Price Drop Alerts with Email Notifications and Database Tracking

### 1. Workflow Overview

This workflow, titled **"Automated Hotel Price Drop Alerts with Email Notifications and Database Tracking"**, automates the monitoring of hotel room prices across multiple properties. It periodically checks current rates via APIs, compares them against historical data to detect price drops, and sends styled email alerts to a travel agent when savings are found. It also logs all checks and updates a database with the latest price information for future comparisons.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Initiates the price check every 6 hours automatically.
- **1.2 Hotel List Loading:** Loads the list of hotels and room types to monitor.
- **1.3 Price Retrieval and Parsing:** Fetches current prices from external APIs and extracts relevant pricing and availability data.
- **1.4 Historical Price Lookup:** Queries the database for previously recorded prices to enable comparisons.
- **1.5 Price Comparison Logic:** Calculates price differences, percentage changes, and determines if an alert is necessary.
- **1.6 Alert Decision and Formatting:** Filters for price reductions, formats a detailed HTML email alert with booking details and savings.
- **1.7 Email Delivery:** Sends the alert email to the travel agent.
- **1.8 Logging and Database Update:** Logs both alert and no-alert outcomes and updates the price record in the database.
- **1.9 Execution Summary:** Generates an overall summary report of the execution for monitoring purposes.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Automatically triggers the workflow every 6 hours to perform price monitoring checks.
- **Nodes Involved:**  
  - Schedule - Every 6 Hours  
  - Sticky Note (trigger explanation)
- **Node Details:**  
  - **Schedule - Every 6 Hours**  
    - Type: Schedule Trigger  
    - Configuration: Interval set to every 6 hours  
    - Output: Triggers the next node "Load Hotel List"  
    - Edge cases: Workflow won't trigger if n8n instance is down; no retries on missed triggers by default.  
  - **Sticky Note:** Describes the trigger purpose; no functional impact.

---

#### 1.2 Hotel List Loading

- **Overview:** Loads a predefined list of hotels and room types to monitor, including metadata such as API endpoints and booking URLs.
- **Nodes Involved:**  
  - Load Hotel List (Code)  
  - Sticky Note (hotel list explanation)
- **Node Details:**  
  - **Load Hotel List**  
    - Type: Code node (JavaScript)  
    - Configuration: Hardcoded array of 5 hotels with details (hotelId, name, location, roomType, API endpoint, booking URL)  
    - Adds timestamps for check time and date  
    - Output: JSON items representing hotels, passed to "Fetch Current Price from API"  
    - Edge cases: In production, this should query a database; hardcoding limits scalability and dynamic updates.  
  - **Sticky Note:** Describes loading hotels for monitoring.

---

#### 1.3 Price Retrieval and Parsing

- **Overview:** Fetches current pricing data from hotel APIs and parses the response to extract price, availability, and other relevant info.
- **Nodes Involved:**  
  - Fetch Current Price from API (HTTP Request)  
  - Parse Price Data (Code)  
  - Sticky Notes for API fetch and parsing
- **Node Details:**  
  - **Fetch Current Price from API**  
    - Type: HTTP Request  
    - Configuration: GET request to dynamic URL from hotel item JSON (`apiEndpoint`), with JSON headers  
    - Options: Never error on response to avoid workflow failure if API issues  
    - Output: API response forwarded to "Parse Price Data"  
    - Edge cases: API downtime, unexpected response format, authentication (not configured here), latency/timeouts  
  - **Parse Price Data**  
    - Type: Code Node  
    - Configuration: Simulates API response by generating random price variations and availability status  
    - Extracts and formats data fields (currentPrice, currency, availability, taxesIncluded, timestamps)  
    - Output: Structured JSON for next steps  
    - Edge cases: Assumes API response structure; simulation means actual API parsing logic must be implemented in production  
  - **Sticky Notes:** Explain purpose of each node.

---

#### 1.4 Historical Price Lookup

- **Overview:** Queries a database API to retrieve the last known price for the hotel and room type for comparison.
- **Nodes Involved:**  
  - Get Previous Price from DB (HTTP Request)  
  - Sticky Note (database query explanation)
- **Node Details:**  
  - **Get Previous Price from DB**  
    - Type: HTTP Request  
    - Configuration: GET request with query parameters for hotel_id and room_type from parsed data  
    - Options: Never error on response to handle missing data gracefully  
    - Output: Previous price data (simulated in code later)  
    - Edge cases: Network issues, database API downtime, missing previous records  
  - **Sticky Note:** Describes database query for previous price.

---

#### 1.5 Price Comparison Logic

- **Overview:** Compares current price with previous price, calculates difference, percentage change, and determines if price changed or reduced.
- **Nodes Involved:**  
  - Compare Prices (Code)  
  - Sticky Note (comparison explanation)
- **Node Details:**  
  - **Compare Prices**  
    - Type: Code Node  
    - Configuration: Simulates previous price with random variation (for demo); calculates difference, percentage change, flags for priceReduced, priceIncreased, priceChanged  
    - Outputs enriched JSON with comparison and savings data for alert logic  
    - Edge cases: Previous price zero or null handled by zero percentage change fallback  
  - **Sticky Note:** Explains price comparison calculations.

---

#### 1.6 Alert Decision and Formatting

- **Overview:** Filters only cases where price has decreased and prepares a visually formatted HTML email alert including savings, booking link, and urgency.
- **Nodes Involved:**  
  - Check if Price Reduced (If)  
  - Format Alert Email (Code)  
  - Sticky Notes (filter and email formatting)
- **Node Details:**  
  - **Check if Price Reduced**  
    - Type: If Node  
    - Configuration: Condition checks if `priceReduced` is true  
    - Outputs:  
      - True branch to "Format Alert Email"  
      - False branch to "Log No Alert Needed"  
    - Edge cases: Misconfigured condition could send unnecessary alerts or none at all  
  - **Format Alert Email**  
    - Type: Code Node  
    - Configuration: Builds detailed HTML email with inline CSS, showing previous vs current price with styling, savings, urgency levels, booking link, and disclaimers  
    - Sets email subject dynamically including hotel name and savings amount  
    - Output: JSON with email content for sending  
    - Edge cases: HTML rendering issues in email clients, encoding special characters  
  - **Sticky Notes:** Describe purpose of filtering and email formatting.

---

#### 1.7 Email Delivery

- **Overview:** Sends the prepared email alert to the travel agent using SMTP.
- **Nodes Involved:**  
  - Send Email to Travel Agent (Email Send)  
  - Sticky Note (email delivery explanation)
- **Node Details:**  
  - **Send Email to Travel Agent**  
    - Type: Email Send  
    - Configuration: Uses SMTP credentials, sender set to alerts@hotelpricemonitor.com, recipient travelagent@agency.com, subject and body from previous node  
    - Output: Passes data to "Log Alert Sent"  
    - Edge cases: SMTP authentication failure, email delivery failure, spam filtering  
  - **Sticky Note:** Explains email delivery.

---

#### 1.8 Logging and Database Update

- **Overview:** Logs both alert and no-alert scenarios for audit and tracking; updates database with the latest price.
- **Nodes Involved:**  
  - Log Alert Sent (Code)  
  - Log No Alert Needed (Code)  
  - Merge All Logs (Merge)  
  - Update Price in Database (HTTP Request)  
  - Sticky Notes (logging and DB update)
- **Node Details:**  
  - **Log Alert Sent**  
    - Type: Code Node  
    - Configuration: Records alert details including hotel info, alert timestamp, savings, recipient email  
  - **Log No Alert Needed**  
    - Type: Code Node  
    - Configuration: Records cases where price did not decrease, including reason (price increased or unchanged)  
  - **Merge All Logs**  
    - Type: Merge Node  
    - Configuration: Combines logs from alert and no-alert paths for unified downstream processing  
  - **Update Price in Database**  
    - Type: HTTP Request  
    - Configuration: POST request with JSON body updating hotel_id, room_type, current price, currency, availability, and timestamp  
    - Headers: Content-Type application/json  
    - Edge cases: Database write failure, API downtime  
  - **Sticky Notes:** Explain logging and database update roles.

---

#### 1.9 Execution Summary

- **Overview:** Generates a summary report of the entire execution including counts of hotels checked, alerts sent, and next scheduled check time.
- **Nodes Involved:**  
  - Create Execution Summary (Code)  
  - Sticky Note (summary explanation)
- **Node Details:**  
  - **Create Execution Summary**  
    - Type: Code Node  
    - Configuration: Aggregates all logs to compute total hotels checked, alerts sent, alert rate percentage, and timestamps; outputs a JSON summary  
    - Output: Can be used for reporting or further notifications  
  - **Sticky Note:** Describes execution summary purpose.

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                              | Input Node(s)               | Output Node(s)               | Sticky Note                                                |
|------------------------------|-----------------------|----------------------------------------------|-----------------------------|------------------------------|------------------------------------------------------------|
| Schedule - Every 6 Hours      | Schedule Trigger      | Initiates workflow every 6 hours             |                             | Load Hotel List              | Triggers price check automatically every 6 hours           |
| Sticky Note                  | Sticky Note           | Describes trigger                            |                             |                              | ## ⏰ TRIGGER Runs price check every 6 hours automatically  |
| Load Hotel List              | Code                  | Loads hotels list to monitor                  | Schedule - Every 6 Hours     | Fetch Current Price from API | ## 🏨 HOTEL LIST Loads all hotels that need price monitoring |
| Sticky Note1                 | Sticky Note           | Describes hotel list loading                  |                             |                              |                                                            |
| Fetch Current Price from API | HTTP Request          | Fetches current prices from hotel APIs       | Load Hotel List              | Parse Price Data             | ## 💰 PRICE API Fetches current room rates in real-time    |
| Sticky Note2                 | Sticky Note           | Explains price API fetch                      |                             |                              |                                                            |
| Parse Price Data             | Code                  | Parses and formats API price data             | Fetch Current Price from API | Get Previous Price from DB   | ## 📊 PARSE DATA Extracts price and availability info      |
| Sticky Note3                 | Sticky Note           | Describes parsing of price data               |                             |                              |                                                            |
| Get Previous Price from DB   | HTTP Request          | Retrieves previous price from database        | Parse Price Data             | Compare Prices              | ## 🗄️ DATABASE QUERY Fetches previous price record          |
| Sticky Note4                 | Sticky Note           | Describes database query                      |                             |                              |                                                            |
| Compare Prices              | Code                  | Compares current vs previous prices            | Get Previous Price from DB   | Check if Price Reduced       | ## 🔍 PRICE COMPARISON Calculates difference and change %  |
| Sticky Note5                 | Sticky Note           | Explains price comparison                      |                             |                              |                                                            |
| Check if Price Reduced       | If                    | Filters only price reductions                   | Compare Prices              | Format Alert Email / Log No Alert Needed | ## ✅ PRICE FILTER Identifies reduced prices only           |
| Sticky Note6                 | Sticky Note           | Describes filtering for reduced prices         |                             |                              |                                                            |
| Format Alert Email           | Code                  | Creates HTML email alert with details          | Check if Price Reduced (true) | Send Email to Travel Agent  | ## 📧 EMAIL FORMATTING Creates beautiful HTML alert email   |
| Sticky Note7                 | Sticky Note           | Describes email formatting                      |                             |                              |                                                            |
| Send Email to Travel Agent   | Email Send            | Sends alert email to travel agent               | Format Alert Email           | Log Alert Sent              | ## 📮 EMAIL DELIVERY Sends alert to travel agent            |
| Sticky Note8                 | Sticky Note           | Explains email delivery                         |                             |                              |                                                            |
| Log Alert Sent               | Code                  | Logs successful alert delivery                   | Send Email to Travel Agent   | Merge All Logs              | ## 📝 ALERT LOGGING Records successful alert delivery       |
| Sticky Note9                 | Sticky Note           | Describes alert logging                          |                             |                              |                                                            |
| Log No Alert Needed          | Code                  | Logs checks where no alert was needed            | Check if Price Reduced (false) | Merge All Logs              | ## 📋 NO ALERT LOG Records when prices didn't decrease       |
| Sticky Note10                | Sticky Note           | Describes no alert logging                       |                             |                              |                                                            |
| Merge All Logs               | Merge                 | Combines alert and no-alert logs                 | Log Alert Sent, Log No Alert Needed | Update Price in Database    | ## 🔀 LOG MERGE Combines all check results                   |
| Sticky Note11                | Sticky Note           | Explains log merging                             |                             |                              |                                                            |
| Update Price in Database     | HTTP Request          | Updates latest price info in database            | Merge All Logs              | Create Execution Summary    | ## 💾 DATABASE UPDATE Stores new price for next comparison  |
| Sticky Note12                | Sticky Note           | Describes database update                         |                             |                              |                                                            |
| Create Execution Summary     | Code                  | Generates summary report of execution             | Update Price in Database    |                              | ## 📊 SUMMARY REPORT Creates execution statistics            |
| Sticky Note13                | Sticky Note           | Explains summary report                          |                             |                              |                                                            |
| Sticky Note14                | Sticky Note           | Overview of entire workflow features and setup   |                             |                              | ## 🏨 HOTEL ROOM PRICE CHECKER Automated Price Monitoring System |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours  
   - Connect output to "Load Hotel List" node

2. **Create "Load Hotel List" Code Node**  
   - Type: Code Node (JavaScript)  
   - Paste hardcoded array of hotels with keys: hotelId, hotelName, location, roomType, bookingUrl, apiEndpoint  
   - Add current timestamp and formatted date fields for each hotel  
   - Connect output to "Fetch Current Price from API" node

3. **Create "Fetch Current Price from API" HTTP Request Node**  
   - Type: HTTP Request  
   - Set URL to `{{$json.apiEndpoint}}` (dynamic from input)  
   - Method: GET  
   - Headers: Content-Type and Accept as application/json  
   - Enable "Never error on response" to handle API errors gracefully  
   - Connect output to "Parse Price Data" node

4. **Create "Parse Price Data" Code Node**  
   - Type: Code Node  
   - Simulate API response by generating random current price, availability, and currency  
   - Output structured JSON with hotel info, currentPrice, currency, availability, taxesIncluded, and timestamps  
   - Connect output to "Get Previous Price from DB" node

5. **Create "Get Previous Price from DB" HTTP Request Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Your database API endpoint (e.g., https://your-database-api.com/prices/history)  
   - Query Parameters: hotel_id and room_type from input JSON  
   - Enable "Never error on response"  
   - Connect output to "Compare Prices" node

6. **Create "Compare Prices" Code Node**  
   - Type: Code Node  
   - Calculate previous price (simulate or use actual DB response)  
   - Compute priceDifference, percentageChange, priceReduced, priceIncreased, priceChanged, savingsAmount  
   - Add flags and enriched data for alert logic  
   - Connect output to "Check if Price Reduced" node

7. **Create "Check if Price Reduced" If Node**  
   - Type: If Node  
   - Condition: Check if `priceReduced` field is true  
   - True output connects to "Format Alert Email" node  
   - False output connects to "Log No Alert Needed" node

8. **Create "Format Alert Email" Code Node**  
   - Type: Code Node  
   - Build HTML email with inline CSS including hotel details, old/new prices, savings, booking URL, urgency level  
   - Create subject line with hotel name and savings amount  
   - Connect output to "Send Email to Travel Agent" node

9. **Create "Send Email to Travel Agent" Email Send Node**  
   - Type: Email Send  
   - Configure SMTP credentials (e.g., SMTP -test)  
   - From Email: alerts@hotelpricemonitor.com  
   - To Email: travelagent@agency.com  
   - Subject and body from previous node's JSON  
   - Connect output to "Log Alert Sent" node

10. **Create "Log Alert Sent" Code Node**  
    - Type: Code Node  
    - Log alert details: hotelId, hotelName, alertSent=true, alertTimestamp, previousPrice, currentPrice, savingsAmount, recipientEmail, alertType  
    - Connect output to "Merge All Logs" node

11. **Create "Log No Alert Needed" Code Node**  
    - Type: Code Node  
    - Log no alert cases: hotelId, hotelName, alertSent=false, checkTimestamp, previousPrice, currentPrice, priceChanged, changeDirection, reason (price_increased or no_change)  
    - Connect output to "Merge All Logs" node

12. **Create "Merge All Logs" Merge Node**  
    - Type: Merge  
    - Mode: Combine  
    - Input from both "Log Alert Sent" and "Log No Alert Needed"  
    - Connect output to "Update Price in Database" node

13. **Create "Update Price in Database" HTTP Request Node**  
    - Type: HTTP Request  
    - Method: POST or PUT depending on your API  
    - URL: Your database API endpoint for updating prices (e.g., https://your-database-api.com/prices/update)  
    - Body Parameters: hotel_id, room_type, price, currency, availability, timestamp from parsed price data  
    - Headers: Content-Type application/json  
    - Connect output to "Create Execution Summary" node

14. **Create "Create Execution Summary" Code Node**  
    - Type: Code Node  
    - Aggregate all logs from previous nodes  
    - Calculate total hotels checked, alerts sent, no alerts needed, alert rate, current timestamp and next check time  
    - Output summary JSON for reporting or monitoring

15. **Add Sticky Notes**  
    - Add descriptive sticky notes at key places for documentation and clarity per the original workflow content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow simulates API responses with random pricing; replace simulation with real API parsing for production.                                                                                                            | Implementation detail                             |
| SMTP credentials must be configured with valid email server details to enable email delivery.                                                                                                                                  | Setup requirement for "Send Email to Travel Agent" |
| Database API endpoints used for fetching and updating prices are placeholders; adapt to your actual database service or use n8n database nodes if preferred.                                                                  | Integration point                                 |
| The HTML email includes inline CSS for consistent rendering across email clients, including urgency indicators and a booking call-to-action button.                                                                            | Email formatting best practice                     |
| The workflow includes detailed logging for both alert and non-alert cases, which supports audit trails and analytics.                                                                                                          | Operational monitoring                             |
| Alert urgency is set dynamically based on percentage savings (>20% is high urgency).                                                                                                                                           | Business logic detail                              |
| The workflow requires n8n version supporting HTTP Request node v4.2 and Code node v2.                                                                                                                                          | Version compatibility                              |
| For scalability, consider replacing the hardcoded hotel list with a database or API source.                                                                                                                                   | Scalability advice                                |
| Sticky notes provide inline documentation and explanations for maintenance and onboarding.                                                                                                                                     | Documentation best practice                        |
| Overview sticky note summarizes workflow features and setup steps.                                                                                                                                                             | Workflow meta-documentation                        |

---

This document provides a complete understanding and reproduction guide for the "Automated Hotel Price Drop Alerts" workflow in n8n, enabling users to maintain, extend, or integrate it within their environment effectively.