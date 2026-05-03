Gap-Time Cafe Finder with OpenRouter AI, Google Calendar, Maps & Slack Alerts

https://n8nworkflows.xyz/workflows/gap-time-cafe-finder-with-openrouter-ai--google-calendar--maps---slack-alerts-11034


# Gap-Time Cafe Finder with OpenRouter AI, Google Calendar, Maps & Slack Alerts

### 1. Workflow Overview

The **Smart Gap-Time Cafe Concierge with AI Recommendations** workflow aims to optimize short free periods ("gap times") between scheduled events by recommending nearby cafes tailored to user preferences. It integrates Google Calendar, Google Maps, Google Places API, AI-powered recommendation, and Slack notifications to automate the entire process.

**Target Use Cases:**
- Individuals seeking to maximize short free intervals during their day by finding suitable cafes near their current location.
- Users who want personalized cafe recommendations based on their preferences.
- Automated alerts for when time is too short to visit a cafe, ensuring punctuality.

**Logical Blocks:**

- **1.1 Schedule Trigger & Configuration:** Initiates the workflow at a fixed time daily with essential configuration parameters.
- **1.2 Calendar Event & Travel Time Retrieval:** Fetches the next calendar event and calculates travel time to it.
- **1.3 Gap Time Calculation & Conditional Check:** Determines available free time before the event and decides if cafe suggestion is feasible.
- **1.4 User Preferences & Nearby Cafe Search:** Retrieves user cafe preferences and queries nearby cafes.
- **1.5 AI Recommendation:** Uses an AI model to analyze preferences, gap time, and cafe options to select the best recommendation.
- **1.6 Notifications via Slack:** Sends either a cafe recommendation or an urgent move alert to Slack, depending on available gap time.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Schedule Trigger & Configuration

**Overview:**  
This block initializes the workflow daily at noon and sets up crucial variables such as the user's current location and API keys for Google services.

**Nodes Involved:**  
- Schedule Trigger  
- Workflow Configuration

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow automatically at 12:00 PM daily.  
  - Configuration: Trigger set to fire once daily at hour 12 (noon).  
  - Input: None  
  - Output: Workflow Configuration node  
  - Edge Cases: Misconfigured schedule may cause missed triggers; timezone differences can affect timing.

- **Workflow Configuration**  
  - Type: Set  
  - Role: Stores configuration variables essential for downstream nodes.  
  - Configuration:  
    - `currentLocation`: User’s current latitude and longitude or address (placeholder to be replaced).  
    - `googleMapsApiKey`: API key for Google Maps Distance Matrix API.  
    - `googlePlacesApiKey`: API key for Google Places API.  
    - `minimumGapMinutes`: Minimum free time threshold (default 30 minutes).  
  - Input: Schedule Trigger  
  - Output: Get Next Calendar Event  
  - Edge Cases: Missing or invalid API keys or location will cause API call failures.

---

#### 1.2 Calendar Event & Travel Time Retrieval

**Overview:**  
Fetches the next upcoming event from Google Calendar and calculates the public transit travel time from the current location to that event’s location.

**Nodes Involved:**  
- Get Next Calendar Event  
- Get Travel Time (Google Maps API)

**Node Details:**

- **Get Next Calendar Event**  
  - Type: Google Calendar  
  - Role: Retrieves the single next upcoming event starting from the current time.  
  - Configuration:  
    - Calendar: User’s Google Calendar account (`bumpbeck913@gmail.com`).  
    - Limit: 1 event.  
    - TimeMin: Current time (dynamic).  
    - Ordered by start time, recurring events expanded.  
  - Input: Workflow Configuration  
  - Output: Get Travel Time (Google Maps API)  
  - Edge Cases: No upcoming events, calendar permission errors, API quota limits.

- **Get Travel Time (Google Maps API)**  
  - Type: HTTP Request  
  - Role: Calls Google Distance Matrix API to get transit time to event location.  
  - Configuration:  
    - URL dynamically constructed using `currentLocation` and event’s location or summary.  
    - Mode: `transit` (public transport).  
    - API key: `googleMapsApiKey` from configuration.  
    - Response format: JSON.  
  - Input: Get Next Calendar Event  
  - Output: Calculate Gap Time  
  - Edge Cases: Invalid locations, API key restrictions, no transit route available, timeout.

---

#### 1.3 Gap Time Calculation & Conditional Check

**Overview:**  
Calculates the actual free time before the next event, factoring in travel time, and conditionally routes workflow based on whether the gap meets the minimum threshold.

**Nodes Involved:**  
- Calculate Gap Time  
- Check If Gap >= 30 Minutes

**Node Details:**

- **Calculate Gap Time**  
  - Type: Code  
  - Role: Computes minutes of free gap time by subtracting travel time from time until event start.  
  - Key Logic:  
    - Extracts event start time and location from calendar data.  
    - Extracts travel duration in seconds and converts to minutes.  
    - Calculates time difference between now and event start in minutes.  
    - Calculates `gapTimeMinutes = timeUntilEventMinutes - travelTimeMinutes`.  
  - Input: Get Travel Time (Google Maps API) and Get Next Calendar Event  
  - Output: Check If Gap >= 30 Minutes  
  - Edge Cases: Missing event times, travel time zero or missing, event in the past.

- **Check If Gap >= 30 Minutes**  
  - Type: If  
  - Role: Determines if gap time is sufficient for a cafe visit.  
  - Configuration:  
    - Condition: `gapTimeMinutes >= minimumGapMinutes` (30 by default).  
  - Input: Calculate Gap Time  
  - Output True: Get User Preferences  
  - Output False: Send Urgent Move Alert (Slack)  
  - Edge Cases: Non-numeric gap times, expression evaluation errors.

---

#### 1.4 User Preferences & Nearby Cafe Search

**Overview:**  
Retrieves user-specific cafe preferences from Google Sheets and queries nearby cafes using Google Places API.

**Nodes Involved:**  
- Get User Preferences  
- Search Nearby Cafes (Google Places API)

**Node Details:**

- **Get User Preferences**  
  - Type: Google Sheets  
  - Role: Loads user cafe preferences to personalize AI recommendations.  
  - Configuration:  
    - Document ID points to a specific Google Sheet with user preferences.  
    - Sheet name is "シート1" (Sheet 1 in Japanese).  
  - Input: Check If Gap >= 30 Minutes (True)  
  - Output: Search Nearby Cafes (Google Places API)  
  - Edge Cases: Sheet access permissions, empty or malformed data.

- **Search Nearby Cafes (Google Places API)**  
  - Type: HTTP Request  
  - Role: Searches for cafes within 1000 meters of current location.  
  - Configuration:  
    - URL built dynamically with `currentLocation`, radius=1000 meters, type=cafe.  
    - API key: `googlePlacesApiKey`.  
    - Response format: JSON.  
  - Input: Get User Preferences  
  - Output: AI Agent  
  - Edge Cases: API key invalid or throttled, no cafes found, malformed location.

---

#### 1.5 AI Recommendation

**Overview:**  
Leverages an AI agent powered by OpenRouter to analyze gap time, user preferences, and nearby cafes, and produce a tailored cafe recommendation.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Executes AI prompt to select the best cafe recommendation with a reason and Google Maps link.  
  - Configuration:  
    - Input text includes gap time, user preferences (JSON), and top 10 nearby cafes (JSON).  
    - System message instructs AI to act as a skilled cafe concierge and output JSON with keys: `cafeName`, `rating`, `reason` (in Japanese), `mapsUrl`.  
  - Input: Search Nearby Cafes (Google Places API), OpenRouter Chat Model  
  - Output: Send Slack Notification  
  - Edge Cases: AI model downtime, malformed input causing prompt errors, rate limits.

- **OpenRouter Chat Model**  
  - Type: LangChain LM Chat OpenRouter  
  - Role: Provides the language model backend for the AI Agent.  
  - Configuration: None directly (API credentials configured externally).  
  - Input: None other than internal from AI Agent  
  - Output: AI Agent  
  - Edge Cases: Credential misconfiguration, API errors, network issues.

---

#### 1.6 Notifications via Slack

**Overview:**  
Sends Slack messages with either the recommended cafe details or an urgent alert to move immediately depending on gap time sufficiency.

**Nodes Involved:**  
- Send Slack Notification  
- Send Urgent Move Alert (Slack)

**Node Details:**

- **Send Slack Notification**  
  - Type: Slack  
  - Role: Notifies user with the cafe recommendation details.  
  - Configuration:  
    - Channel ID: Replace placeholder with actual Slack channel ID (e.g., `#general`).  
    - Text: Dynamic message including gap time, cafe name, rating, recommendation reason, Google Maps URL, next event name, start time, and travel time.  
    - Authentication: OAuth2 with Slack credentials.  
  - Input: AI Agent  
  - Output: None  
  - Edge Cases: Slack API authentication failure, invalid channel ID, message formatting errors.

- **Send Urgent Move Alert (Slack)**  
  - Type: Slack  
  - Role: Sends urgent alert if gap time is insufficient for a cafe visit.  
  - Configuration:  
    - Channel ID: Same as above, replace with actual Slack channel.  
    - Text: Warning message displaying remaining gap time, travel time, and next event details.  
    - Authentication: OAuth2 with Slack credentials.  
  - Input: Check If Gap >= 30 Minutes (False)  
  - Output: None  
  - Edge Cases: Same as above, plus possible repeated alerts if workflow runs frequently.

---

### 3. Summary Table

| Node Name                        | Node Type                          | Functional Role                                   | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                              |
|---------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                | Schedule Trigger                 | Triggers workflow daily at 12 PM                 | None                          | Workflow Configuration                 | This node initiates the workflow at a scheduled time. The workflow is set to run daily at 12 PM (noon).                    |
| Workflow Configuration          | Set                              | Stores current location and API keys              | Schedule Trigger              | Get Next Calendar Event                | Sets core variables: currentLocation, Google API keys, and minimumGapMinutes (30).                                        |
| Get Next Calendar Event         | Google Calendar                  | Fetches next upcoming calendar event              | Workflow Configuration       | Get Travel Time (Google Maps API)     | Connects to Google Calendar to get the next event starting from now.                                                      |
| Get Travel Time (Google Maps API) | HTTP Request                    | Calls Google Distance Matrix API for transit time | Get Next Calendar Event       | Calculate Gap Time                     | Calculates transit time to event location using Google Maps Distance Matrix API.                                          |
| Calculate Gap Time              | Code                             | Calculates available free gap time before event   | Get Travel Time (Google Maps API), Get Next Calendar Event | Check If Gap >= 30 Minutes          | Computes `gapTimeMinutes` by subtracting travel time from time until event start.                                        |
| Check If Gap >= 30 Minutes      | If                               | Checks if gap time is at least minimum threshold  | Calculate Gap Time            | Get User Preferences (True), Send Urgent Move Alert (False) | Conditional gate on gap time to decide next steps.                                                                         |
| Get User Preferences            | Google Sheets                   | Loads user cafe preferences                        | Check If Gap >= 30 Minutes (True) | Search Nearby Cafes (Google Places API) | Retrieves user preferences from Google Sheets for AI recommendation.                                                     |
| Search Nearby Cafes (Google Places API) | HTTP Request               | Finds nearby cafes within 1000m radius             | Get User Preferences          | AI Agent                              | Searches cafes around current location with Google Places API.                                                           |
| AI Agent                       | LangChain AI Agent               | Generates cafe recommendation using AI            | Search Nearby Cafes, OpenRouter Chat Model | Send Slack Notification               | AI concierge selects the best cafe based on gap time, preferences, and nearby cafes.                                      |
| OpenRouter Chat Model           | LangChain LM Chat OpenRouter     | Provides AI language model backend                  | None                         | AI Agent                              | AI model node configured to use OpenRouter API.                                                                           |
| Send Slack Notification         | Slack                           | Sends recommended cafe info to Slack channel       | AI Agent                    | None                                 | Sends cafe recommendation message if gap time is sufficient.                                                             |
| Send Urgent Move Alert (Slack)  | Slack                           | Sends urgent alert if gap time is insufficient      | Check If Gap >= 30 Minutes (False) | None                                 | Sends urgent Slack alert to move immediately if not enough time for cafe visit.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run daily at hour 12 (noon).  
   - No credentials needed.

2. **Create Workflow Configuration Node (Set)**  
   - Connect Schedule Trigger → Workflow Configuration.  
   - Add variables:  
     - `currentLocation`: Your current latitude, longitude, or address (e.g., "35.6812,139.7671").  
     - `googleMapsApiKey`: Your Google Maps API key.  
     - `googlePlacesApiKey`: Your Google Places API key.  
     - `minimumGapMinutes`: Number, default 30.

3. **Create Get Next Calendar Event Node (Google Calendar)**  
   - Connect Workflow Configuration → Get Next Calendar Event.  
   - Set calendar account (OAuth2) with your Google credentials.  
   - Limit: 1 event.  
   - TimeMin: `={{ $now.toISO() }}`.  
   - Options: Order by startTime, expand recurring events.

4. **Create Get Travel Time (Google Maps API) Node (HTTP Request)**  
   - Connect Get Next Calendar Event → Get Travel Time (Google Maps API).  
   - Method: GET.  
   - URL:  
     ```
     =https://maps.googleapis.com/maps/api/distancematrix/json?origins={{ $('Workflow Configuration').first().json.currentLocation }}&destinations={{ encodeURIComponent($json.location || $json.summary) }}&mode=transit&key={{ $('Workflow Configuration').first().json.googleMapsApiKey }}
     ```  
   - Response Format: JSON.

5. **Create Calculate Gap Time Node (Code)**  
   - Connect Get Travel Time → Calculate Gap Time.  
   - Paste provided JavaScript code to calculate gap time (subtracting travel from time until event).  
   - Outputs calculated variables including `gapTimeMinutes`.

6. **Create Check If Gap >= 30 Minutes Node (If)**  
   - Connect Calculate Gap Time → Check If Gap >= 30 Minutes.  
   - Condition: `{{$json.gapTimeMinutes}} >= {{$('Workflow Configuration').first().json.minimumGapMinutes}}`.

7. **Create Get User Preferences Node (Google Sheets)**  
   - Connect Check If Gap >= 30 Minutes (True) → Get User Preferences.  
   - Configure Google Sheets credentials.  
   - Set Spreadsheet ID and Sheet Name (e.g., "シート1").  
   - This sheet contains user cafe preferences.

8. **Create Search Nearby Cafes (Google Places API) Node (HTTP Request)**  
   - Connect Get User Preferences → Search Nearby Cafes (Google Places API).  
   - Method: GET.  
   - URL:  
     ```
     =https://maps.googleapis.com/maps/api/place/nearbysearch/json?location={{ $('Workflow Configuration').first().json.currentLocation }}&radius=1000&type=cafe&key={{ $('Workflow Configuration').first().json.googlePlacesApiKey }}
     ```  
   - Response Format: JSON.

9. **Create OpenRouter Chat Model Node (LangChain LM Chat OpenRouter)**  
   - No direct connection; this node is referenced by AI Agent.  
   - Configure OpenRouter API credentials.

10. **Create AI Agent Node (LangChain AI Agent)**  
    - Connect Search Nearby Cafes → AI Agent.  
    - Configure to use OpenRouter Chat Model node as language model.  
    - Set prompt with: gap time, user preferences JSON, nearby cafes JSON, and instructions to return JSON with keys `cafeName`, `rating`, `reason`, `mapsUrl`.  
    - System message instructs AI to act as cafe concierge.

11. **Create Send Slack Notification Node (Slack)**  
    - Connect AI Agent → Send Slack Notification.  
    - Configure Slack OAuth2 credentials.  
    - Set channel ID to your Slack channel.  
    - Message includes gap time, recommended cafe details, event info, and travel time.

12. **Create Send Urgent Move Alert (Slack) Node (Slack)**  
    - Connect Check If Gap >= 30 Minutes (False) → Send Urgent Move Alert (Slack).  
    - Configure Slack OAuth2 credentials.  
    - Set same Slack channel ID as above.  
    - Message warns user to move immediately, showing remaining time and event details.

13. **Test the workflow end to end**, adjusting API keys, location, and Slack channel IDs as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                  | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow acts as a Smart Gap-Time Cafe Concierge. It automatically checks your Google Calendar for upcoming events, calculates your available "gap time" before the next event, and then suggests a nearby cafe using Google Places API and AI.             | Workflow summary provided in Sticky Note node "Sticky Note"                                                  |
| Adjust the Schedule Trigger if you want the workflow to run multiple times a day or at different times.                                                                                                                                                       | Sticky Note1                                                                                                 |
| Replace placeholders in Workflow Configuration with your actual location and valid Google API keys.                                                                                                                                                          | Sticky Note2                                                                                                 |
| Ensure your Google Maps API key has access to Distance Matrix API; Google Places API key must have Nearby Search permission.                                                                                                                                   | Sticky Note3 and Sticky Note8                                                                                 |
| Google Calendar node requires OAuth2 access to your calendar; make sure the correct account is selected.                                                                                                                                                      | Sticky Note4                                                                                                 |
| The AI Agent uses OpenRouter API; ensure your OpenRouter API key is configured in n8n credentials.                                                                                                                                                            | Sticky Note9 and Sticky Note10                                                                                |
| Slack nodes require OAuth2 authentication; replace the Slack channel IDs with your actual channel where you want notifications.                                                                                                                               | Sticky Note11 and Sticky Note12                                                                               |
| The AI prompt expects user preferences formatted as JSON and up to 10 nearby cafes; keep Google Sheets data structured accordingly.                                                                                                                          | AI Agent node details                                                                                         |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data manipulated is legal and publicly available.