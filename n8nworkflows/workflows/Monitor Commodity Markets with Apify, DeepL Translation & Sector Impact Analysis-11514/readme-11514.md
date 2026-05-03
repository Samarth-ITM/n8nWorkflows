Monitor Commodity Markets with Apify, DeepL Translation & Sector Impact Analysis

https://n8nworkflows.xyz/workflows/monitor-commodity-markets-with-apify--deepl-translation---sector-impact-analysis-11514


# Monitor Commodity Markets with Apify, DeepL Translation & Sector Impact Analysis

---

## 1. Workflow Overview

This workflow is an automated real-time alert system that notifies users when the International Space Station (ISS) is passing overhead their specified location. It only sends alerts when weather conditions are favorable for observing the ISS. The notifications are dispatched via multiple communication channels including Discord, Telegram, and Gmail.

### Target Use Cases:
- Space and astronomy enthusiasts who want timely ISS sighting alerts.
- Educators or parents seeking engaging science moments.
- Remote workers looking for fun reminders or breaks.
- Anyone interested in spotting the ISS with optimal timing and conditions.

### Logical Blocks:

- **1.1 Schedule & ISS Position Retrieval:** Periodically triggers the workflow and fetches the current ISS geographic coordinates.
- **1.2 Distance & Direction Calculation:** Computes the ISS's distance, bearing, direction, and elevation relative to the user’s location.
- **1.3 ISS Overhead Check:** Determines if the ISS is within a user-defined visibility radius.
- **1.4 Weather Condition Analysis:** Retrieves local weather data and assesses observation feasibility.
- **1.5 Observation Feasibility Decision:** Checks if observing the ISS is possible based on weather and time.
- **1.6 Alert Message Formatting:** Creates customized alert messages for good or poor observation conditions.
- **1.7 Crew Information Enrichment:** Adds current ISS crew details to the alert message.
- **1.8 Multi-Channel Notification Dispatch:** Sends the enriched alert messages to Discord, Telegram, and Gmail.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule & ISS Position Retrieval

**Overview:**  
This block triggers the workflow every 10 minutes and requests the current ISS position from a public API.

**Nodes Involved:**  
- Schedule Trigger  
- Get ISS Position

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodically initiates the workflow every 10 minutes.  
  - Configuration: Interval set to every 10 minutes.  
  - Inputs: None (start node)  
  - Outputs: Triggers “Get ISS Position”  
  - Edge Cases: Workflow might miss rapid ISS passes if interval is too long; network errors on trigger unlikely.

- **Get ISS Position**  
  - Type: HTTP Request  
  - Role: Fetches current ISS latitude and longitude from Open Notify API (http://api.open-notify.org/iss-now.json).  
  - Configuration: Simple GET request, no authentication.  
  - Inputs: From Schedule Trigger  
  - Outputs: JSON with ISS position and timestamp to “Calculate Distance and Direction”  
  - Edge Cases: API downtime, slow response, malformed JSON.

---

### 2.2 Distance & Direction Calculation

**Overview:**  
Calculates the distance, compass bearing, direction name, and elevation angle of the ISS relative to the user’s configured location. Also determines if the ISS is within visibility radius.

**Nodes Involved:**  
- Calculate Distance and Direction

**Node Details:**

- **Calculate Distance and Direction**  
  - Type: Code (JavaScript)  
  - Role:  
    - Uses Haversine formula to compute distance between ISS and user's coordinates.  
    - Calculates bearing and converts it to a compass direction (e.g., North, Northeast).  
    - Estimates ISS elevation angle above horizon.  
    - Compares distance to visibility radius to flag if ISS is overhead.  
  - Configuration:  
    - User must set `USER_LAT`, `USER_LON`, `USER_LOCATION_NAME`, and `VISIBILITY_RADIUS_KM` variables inside the code.  
  - Key Expressions: Custom JS functions for geospatial calculations.  
  - Inputs: JSON from “Get ISS Position”  
  - Outputs: JSON including distance, direction, elevation, and visibility boolean to “Is ISS Overhead?”  
  - Edge Cases:  
    - Misconfiguration of user location parameters.  
    - Invalid or missing ISS data.  
  - Version Requirement: Code node v2 used (ensures modern JS support).

---

### 2.3 ISS Overhead Check

**Overview:**  
Determines whether the ISS is currently within the configured visibility radius and branches the flow accordingly.

**Nodes Involved:**  
- Is ISS Overhead?  
- ISS Not Overhead (Set node)

**Node Details:**

- **Is ISS Overhead?**  
  - Type: If  
  - Role: Checks if `isNearby` boolean from previous node is true.  
  - Configuration: Strict boolean equals condition comparing `$json.isNearby` to true.  
  - Inputs: From “Calculate Distance and Direction”  
  - Outputs:  
    - True branch: Proceeds to weather check.  
    - False branch: Proceeds to “ISS Not Overhead” node (no alert).  
  - Edge Cases: Incorrect boolean type or missing `isNearby` value.

- **ISS Not Overhead**  
  - Type: Set  
  - Role: Placeholder node that does nothing, effectively ends the workflow path when ISS is not overhead.  
  - Inputs: From “Is ISS Overhead?” false branch  
  - Outputs: None (ends flow)  
  - Edge Cases: None.

---

### 2.4 Weather Condition Analysis

**Overview:**  
Fetches current local weather data and evaluates whether the ISS can be observed, considering cloud cover, precipitation, and time of day.

**Nodes Involved:**  
- Get Local Weather  
- Analyze Observation Conditions

**Node Details:**

- **Get Local Weather**  
  - Type: HTTP Request  
  - Role: Requests weather data from OpenWeatherMap API using user coordinates.  
  - Configuration:  
    - URL built dynamically with user latitude, longitude, API key, units in metric, language English.  
    - Uses stored OpenWeatherMap API credentials.  
  - Inputs: From “Is ISS Overhead?” true branch  
  - Outputs: JSON weather data to “Analyze Observation Conditions”  
  - Edge Cases: API key errors, rate limits, network issues.

- **Analyze Observation Conditions**  
  - Type: Code (JavaScript)  
  - Role:  
    - Extracts weather main condition, cloudiness, visibility.  
    - Assigns observation condition: "good", "fair", or "poor".  
    - Generates advice messages based on weather and time of day.  
    - Determines final boolean `canObserve`.  
  - Inputs: Weather JSON  
  - Outputs: Enhanced JSON with observation feasibility and advice to “Can Observe?” node  
  - Edge Cases: Missing weather fields, unexpected API responses.

---

### 2.5 Observation Feasibility Decision

**Overview:**  
Decides whether to proceed with alert formatting based on observation feasibility.

**Nodes Involved:**  
- Can Observe? (If node)

**Node Details:**

- **Can Observe?**  
  - Type: If  
  - Role: Checks if `canObserve` is true to choose alert formatting path.  
  - Configuration: Boolean equals comparison on `$json.canObserve`.  
  - Inputs: From “Analyze Observation Conditions”  
  - Outputs:  
    - True branch: “Format Alert (Good Conditions)”  
    - False branch: “Format Alert (Poor Conditions)”  
  - Edge Cases: Missing or invalid `canObserve` value.

---

### 2.6 Alert Message Formatting

**Overview:**  
Formats notification messages differently depending on observation conditions.

**Nodes Involved:**  
- Format Alert (Good Conditions)  
- Format Alert (Poor Conditions)  
- Merge Messages

**Node Details:**

- **Format Alert (Good Conditions)**  
  - Type: Code (JavaScript)  
  - Role: Creates a detailed notification emphasizing good viewing conditions, including how to spot the ISS and weather info.  
  - Outputs JSON with keys: `notificationMessage`, `shortMessage`, and `notificationType` = 'observation_possible'.  
  - Inputs: From “Can Observe?” true branch.

- **Format Alert (Poor Conditions)**  
  - Type: Code (JavaScript)  
  - Role: Creates a notification indicating poor or difficult observation conditions with weather explanation.  
  - Outputs JSON with similar keys but `notificationType` = 'observation_difficult'.  
  - Inputs: From “Can Observe?” false branch.

- **Merge Messages**  
  - Type: Merge  
  - Role: Combines output from either formatting node into a single stream for enrichment.  
  - Inputs: Both “Format Alert (Good Conditions)” and “Format Alert (Poor Conditions)”  
  - Outputs: To “Get ISS Crew Info”  
  - Edge Cases: Ensure only one branch active at a time to avoid merge conflicts.

---

### 2.7 Crew Information Enrichment

**Overview:**  
Fetches current ISS crew data and appends astronaut names to the notification message.

**Nodes Involved:**  
- Get ISS Crew Info  
- Add Crew Information

**Node Details:**

- **Get ISS Crew Info**  
  - Type: HTTP Request  
  - Role: Calls Open Notify API (http://api.open-notify.org/astros.json) to get people currently in space.  
  - Inputs: From “Merge Messages”  
  - Outputs: JSON with astronaut list to “Add Crew Information”  
  - Edge Cases: API failures or empty crew list.

- **Add Crew Information**  
  - Type: Code (JavaScript)  
  - Role: Filters astronauts aboard ISS and appends their names to the alert message.  
  - Outputs enriched message with astronaut count and names.  
  - Inputs: From “Get ISS Crew Info”  
  - Edge Cases: No astronauts aboard, malformed data.

---

### 2.8 Multi-Channel Notification Dispatch

**Overview:**  
Sends the final enriched notification message to Discord, Telegram, and Gmail channels.

**Nodes Involved:**  
- Send to Discord  
- Send to Telegram  
- Send Email via Gmail

**Node Details:**

- **Send to Discord**  
  - Type: Discord  
  - Role: Sends notification message via Discord webhook.  
  - Configuration: Uses webhook authentication; message content from enriched JSON.  
  - Inputs: From “Add Crew Information”  
  - Edge Cases: Invalid webhook URL, network issues.

- **Send to Telegram**  
  - Type: Telegram  
  - Role: Sends notification to specified Telegram chat.  
  - Configuration: Requires bot authentication and chat ID. Message text from enriched JSON.  
  - Inputs: From “Add Crew Information”  
  - Edge Cases: Invalid token/chat ID, API limits.

- **Send Email via Gmail**  
  - Type: Gmail  
  - Role: Sends email notification with enriched alert as message body.  
  - Configuration: Requires Gmail OAuth2 credentials; recipient email set in parameters.  
  - Inputs: From “Add Crew Information”  
  - Edge Cases: Auth errors, quota limits.

---

## 3. Summary Table

| Node Name                     | Node Type               | Functional Role                            | Input Node(s)                     | Output Node(s)                                  | Sticky Note                                                                                     |
|-------------------------------|-------------------------|--------------------------------------------|----------------------------------|------------------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger        | Initiates workflow every 10 minutes         | None                             | Get ISS Position                               | Step 1: Schedule & Get ISS Position. Runs every 10 minutes to check ISS location.               |
| Get ISS Position              | HTTP Request            | Fetches current ISS coordinates              | Schedule Trigger                 | Calculate Distance and Direction               | Step 1: Schedule & Get ISS Position. Runs every 10 minutes to check ISS location.               |
| Calculate Distance and Direction | Code                   | Calculates distance, bearing, elevation      | Get ISS Position                | Is ISS Overhead?                               | Step 2: Calculate Distance & Direction. Configure user location and alert radius in code node.  |
| Is ISS Overhead?              | If                      | Checks if ISS is within visibility radius    | Calculate Distance and Direction | Get Local Weather / ISS Not Overhead           |                                                                                                |
| ISS Not Overhead              | Set                     | Terminates flow if ISS not nearby             | Is ISS Overhead? (false)        | None                                           |                                                                                                |
| Get Local Weather             | HTTP Request            | Retrieves local weather data                   | Is ISS Overhead? (true)         | Analyze Observation Conditions                 | Step 3: Weather Check. Requires OpenWeatherMap API key.                                        |
| Analyze Observation Conditions | Code                   | Assesses observation feasibility from weather | Get Local Weather               | Can Observe?                                   | Step 3: Weather Check. Requires OpenWeatherMap API key.                                        |
| Can Observe?                 | If                      | Determines if ISS can be observed              | Analyze Observation Conditions  | Format Alert (Good Conditions) / Format Alert (Poor Conditions) |                                                                                                |
| Format Alert (Good Conditions) | Code                   | Formats alert message for good observation     | Can Observe? (true)             | Merge Messages                                 |                                                                                                |
| Format Alert (Poor Conditions) | Code                   | Formats alert message for poor observation     | Can Observe? (false)            | Merge Messages                                 |                                                                                                |
| Merge Messages               | Merge                   | Combines alert messages into single stream    | Format Alert (Good Conditions), Format Alert (Poor Conditions) | Get ISS Crew Info                           |                                                                                                |
| Get ISS Crew Info            | HTTP Request            | Fetches current ISS crew list                   | Merge Messages                 | Add Crew Information                           |                                                                                                |
| Add Crew Information         | Code                    | Adds astronaut names to alert message          | Get ISS Crew Info              | Send to Discord, Send to Telegram, Send Email via Gmail |                                                                                                |
| Send to Discord              | Discord                 | Sends alert to Discord channel via webhook     | Add Crew Information           | None                                           | Step 4: Multi-Channel Notifications. Configure Discord webhook URL.                            |
| Send to Telegram             | Telegram                | Sends alert to Telegram chat                     | Add Crew Information           | None                                           | Step 4: Multi-Channel Notifications. Configure Telegram bot token & chat ID.                   |
| Send Email via Gmail         | Gmail                   | Sends alert email via Gmail                      | Add Crew Information           | None                                           | Step 4: Multi-Channel Notifications. Connect Gmail account.                                   |
| Sticky Note                  | Sticky Note             | Workflow description and setup instructions     | None                          | None                                           | Contains detailed workflow overview and setup instructions.                                   |
| Sticky Note1                 | Sticky Note             | Notes on schedule and ISS position retrieval    | None                          | None                                           | Step 1: Schedule & Get ISS Position explanation.                                              |
| Sticky Note2                 | Sticky Note             | Notes on distance and direction calculation     | None                          | None                                           | Step 2: Configuration instructions for user location and radius in code node.                 |
| Sticky Note3                 | Sticky Note             | Notes on weather data usage and API requirement | None                          | None                                           | Step 3: Weather Check explanation and OpenWeatherMap API key requirement.                      |
| Sticky Note4                 | Sticky Note             | Notes on notification channel configuration      | None                          | None                                           | Step 4: Multi-Channel Notifications setup instructions.                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to every 10 minutes.

2. **Create an HTTP Request node named “Get ISS Position”:**  
   - URL: http://api.open-notify.org/iss-now.json  
   - Method: GET  
   - Connect Schedule Trigger to this node.

3. **Create a Code node named “Calculate Distance and Direction”:**  
   - Paste the provided JavaScript code that:  
     - Defines user latitude (`USER_LAT`), longitude (`USER_LON`), user location name (`USER_LOCATION_NAME`), and visibility radius (`VISIBILITY_RADIUS_KM`).  
     - Calculates distance, bearing, direction, elevation, and whether ISS is nearby.  
   - Connect “Get ISS Position” to this node.

4. **Create an If node named “Is ISS Overhead?”:**  
   - Condition: Check if `isNearby` (from previous node JSON) equals `true` (boolean).  
   - Connect “Calculate Distance and Direction” to this node.

5. **Create a Set node named “ISS Not Overhead”:**  
   - No parameters needed; used to end the workflow path.  
   - Connect “Is ISS Overhead?” false output here.

6. **Create an HTTP Request node named “Get Local Weather”:**  
   - URL expression:  
     `https://api.openweathermap.org/data/2.5/weather?lat={{ $json.userLatitude }}&lon={{ $json.userLongitude }}&appid={{$credentials.openWeatherMapApi.apiKey}}&units=metric&lang=en`  
   - Use OpenWeatherMap API credentials (set up API key in credentials).  
   - Connect “Is ISS Overhead?” true output to this node.

7. **Create a Code node named “Analyze Observation Conditions”:**  
   - Paste the JavaScript code analyzing weather conditions and time to determine observation feasibility and advice.  
   - Connect “Get Local Weather” to this node.

8. **Create an If node named “Can Observe?”:**  
   - Condition: Check if `canObserve` equals `true` (boolean).  
   - Connect “Analyze Observation Conditions” to this node.

9. **Create two Code nodes for alert formatting:**  
   - “Format Alert (Good Conditions)”: formats message for good observation conditions (use provided JS code).  
   - “Format Alert (Poor Conditions)”: formats message for poor observation conditions (use provided JS code).  
   - Connect “Can Observe?” true output to “Format Alert (Good Conditions)”.  
   - Connect “Can Observe?” false output to “Format Alert (Poor Conditions)”.

10. **Create a Merge node named “Merge Messages”:**  
    - Mode: Combine  
    - Connect outputs of both formatting nodes to this merge node.

11. **Create an HTTP Request node named “Get ISS Crew Info”:**  
    - URL: http://api.open-notify.org/astros.json  
    - Connect “Merge Messages” to this node.

12. **Create a Code node named “Add Crew Information”:**  
    - Paste JavaScript code that filters ISS crew and appends their names to the message.  
    - Connect “Get ISS Crew Info” to this node.

13. **Create notification nodes:**  
    - **Discord:** Configure with webhook URL, content set to `={{ $json.enrichedMessage }}`  
    - **Telegram:** Configure with bot token and chat ID, text set to `={{ $json.enrichedMessage }}`  
    - **Gmail:** Configure with Gmail OAuth2 credentials, recipient email address, message set to `={{ $json.enrichedMessage }}`, subject prefixed with ISS alert and user location.  
    - Connect “Add Crew Information” output to each notification node in parallel.

14. **Configure all credentials:**  
    - OpenWeatherMap API key for weather data node.  
    - Discord webhook URL for Discord node.  
    - Telegram bot token and chat ID for Telegram node.  
    - Gmail OAuth2 credentials for Gmail node.

15. **Set user location and visibility radius:**  
    - Edit the “Calculate Distance and Direction” code node to input your latitude, longitude, location name, and desired visibility radius in km.

16. **Activate the workflow.**  
    - Confirm all nodes connected properly.  
    - Test by running manually or waiting for scheduled trigger.

---

## 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Real-time ISS Overhead Alert with Weather Check & Multi-Channel Notifications                                              | Workflow description sticky note inside the workflow                                           |
| Open Notify API for ISS position and crew data (free, no API key required)                                                  | http://api.open-notify.org/                                                                     |
| OpenWeatherMap API required for weather condition checks (free tier available)                                             | https://openweathermap.org/api                                                                  |
| Notification channel setup instructions: Discord webhook, Telegram bot token & chat ID, Gmail OAuth2                        | See Sticky Note4 in workflow                                                                    |
| Haversine formula and bearing calculation used for precise geospatial computations                                          | Implemented in “Calculate Distance and Direction” code node                                     |
| Weather condition evaluated includes main weather, cloudiness, visibility, and time of day for observation feasibility     | “Analyze Observation Conditions” code node                                                     |
| Google Earth link generated dynamically for real-time ISS location visualization                                           | Field `googleEarthUrl` in “Calculate Distance and Direction” node output                        |

---

**Disclaimer:**  
The content is derived exclusively from an n8n automated workflow and respects all content policies. No illegal, offensive, or protected data is included. All data processed is public and legal.

---