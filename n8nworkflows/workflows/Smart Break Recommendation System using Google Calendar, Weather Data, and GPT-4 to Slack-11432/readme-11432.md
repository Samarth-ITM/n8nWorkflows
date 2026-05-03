Smart Break Recommendation System using Google Calendar, Weather Data, and GPT-4 to Slack

https://n8nworkflows.xyz/workflows/smart-break-recommendation-system-using-google-calendar--weather-data--and-gpt-4-to-slack-11432


# Smart Break Recommendation System using Google Calendar, Weather Data, and GPT-4 to Slack

### 1. Workflow Overview

This workflow, titled **Smart Break Recommendation System using Google Calendar, Weather Data, and GPT-4 to Slack**, is designed to automatically identify optimal break spots (cafés or similar venues) during free gaps between scheduled calendar events. It targets busy professionals who want to efficiently use their downtime by considering calendar gaps, travel time, weather conditions, and personal preferences to recommend suitable nearby locations. The personalized recommendations are then sent to Slack as friendly notifications.

**Logical Blocks:**

- **1.1 Input & Configuration Setup:** Initialize scheduled execution and configuration parameters including location, API keys, and gap time thresholds.
- **1.2 Data Retrieval:** Fetch next calendar event and user preferences from Notion.
- **1.3 Environmental Context Acquisition:** Obtain weather data at the event location and calculate travel time via Google Maps.
- **1.4 Gap Time Calculation & Validation:** Compute available free time (gap) and validate if it meets the minimum threshold.
- **1.5 Context Merging & Weather-Based Routing:** Consolidate all contextual data and route to different spot search queries based on weather conditions.
- **1.6 Spot Search & AI Recommendation:** Search for indoor or outdoor spots and leverage GPT-4 to generate personalized top 3 recommendations.
- **1.7 Slack Notification:** Deliver the generated recommendations to a configured Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Configuration Setup

- **Overview:** This block triggers the workflow every 30 minutes and sets all necessary configuration parameters such as current location, API keys, and minimum gap time.
- **Nodes Involved:** Schedule Trigger, Set Configuration
- **Node Details:**

  - **Schedule Trigger**
    - Type: Trigger node (scheduleTrigger)
    - Configuration: Runs every 30 minutes to initiate the workflow.
    - Inputs: None (trigger)
    - Outputs: Connects to Set Configuration
    - Edge cases: If n8n instance is down or paused, triggers will be missed; ensure reliable uptime.

  - **Set Configuration**
    - Type: Set node
    - Configuration: Defines key parameters including:
      - `currentLocation` (default: "Tokyo Station")
      - API keys for Google Maps, Places, OpenWeatherMap
      - Notion database ID
      - `minGapTimeMinutes` (default: 30)
    - Inputs: Trigger from Schedule Trigger
    - Outputs: Connects to fetching user preferences
    - Edge cases: Missing or invalid API keys or database IDs will cause downstream failures.

#### 1.2 Data Retrieval

- **Overview:** Retrieves the next upcoming calendar event and user preferences stored in a Notion database.
- **Nodes Involved:** Get User Preferences from Notion, Get Next Calendar Event
- **Node Details:**

  - **Get User Preferences from Notion**
    - Type: Notion node, resource: databasePage, operation: getAll
    - Configuration: Uses Notion database ID from Set Configuration.
    - Inputs: Set Configuration
    - Outputs: Connects to Get Next Calendar Event
    - Edge cases: Invalid database ID, API rate limits, or connectivity issues may cause failures.

  - **Get Next Calendar Event**
    - Type: Google Calendar node
    - Configuration: Retrieves the next calendar event starting from current time, expands recurring events, limits to 1 event.
    - Inputs: Notion preferences node
    - Outputs: To Get Weather at Destination
    - Edge cases: Empty calendar, no upcoming events, or auth errors can affect output.

#### 1.3 Environmental Context Acquisition

- **Overview:** Obtains weather information at the event location and calculates travel time from the current location using Google Maps Directions API.
- **Nodes Involved:** Get Weather at Destination, Get Travel Time via Google Maps
- **Node Details:**

  - **Get Weather at Destination**
    - Type: OpenWeatherMap node
    - Configuration: Queries weather by city name extracted from the calendar event location.
    - Inputs: Get Next Calendar Event
    - Outputs: To Get Travel Time via Google Maps
    - Edge cases: Missing or invalid location data, API quota exceeded.

  - **Get Travel Time via Google Maps**
    - Type: HTTP Request node
    - Configuration: Calls Google Maps Directions API with parameters:
      - Origin: `currentLocation` from config
      - Destination: Event location from calendar
      - Mode: transit
      - API key from config
    - Inputs: Get Weather at Destination
    - Outputs: To Calculate Available Gap Time
    - Edge cases: Invalid API key, invalid location addresses, no route found.

#### 1.4 Gap Time Calculation & Validation

- **Overview:** Calculates the available free time (gap) between current time and next event start minus travel time. Proceeds only if gap time exceeds configured threshold.
- **Nodes Involved:** Calculate Available Gap Time, Has Sufficient Gap Time?
- **Node Details:**

  - **Calculate Available Gap Time**
    - Type: Code node (JavaScript)
    - Configuration: Computes:
      - Event start time
      - Travel time in minutes from Google Maps response
      - Total available time until event
      - Gap time = total available - travel time
      - Flags if gap time ≥ minimum gap time (from config)
    - Inputs: Get Travel Time via Google Maps, Get Next Calendar Event, Set Configuration
    - Outputs: To If node (Has Sufficient Gap Time?)
    - Edge cases: Incorrect datetime formats, missing travel data cause calculation errors.

  - **Has Sufficient Gap Time?**
    - Type: If node
    - Configuration: Checks if `hasGapTime` is true.
    - Inputs: Calculate Available Gap Time
    - Outputs:
      - True: Proceed to merge context data
      - False: Workflow ends (no further action)
    - Edge cases: If condition fails incorrectly, recommendation might be skipped.

#### 1.5 Context Merging & Weather-Based Routing

- **Overview:** Combines calculated gap time, weather data, user preferences, and calendar event info into a single context object. Routes the workflow based on weather condition to search for indoor or outdoor spots.
- **Nodes Involved:** Merge All Context Data, Route by Weather Condition
- **Node Details:**

  - **Merge All Context Data**
    - Type: Code node (JavaScript)
    - Configuration: Aggregates:
      - Gap time data
      - Weather data (including weather ID)
      - User preferences (all pages from Notion)
      - Calendar event details
      - Configuration parameters
    - Inputs: Has Sufficient Gap Time?
    - Outputs: Route by Weather Condition
    - Edge cases: Missing data from any source may cause incomplete context.

  - **Route by Weather Condition**
    - Type: Switch node
    - Configuration: Routes based on weather ID:
      - `< 700`: Rain/Snow → indoor spots
      - `≥ 700`: Sunny/Cloudy → outdoor spots
    - Inputs: Merge All Context Data
    - Outputs: To Search Indoor Spots or Search Outdoor Spots
    - Edge cases: Incorrect weather ID may route incorrectly.

#### 1.6 Spot Search & AI Recommendation

- **Overview:** Searches for spots near event location based on weather routing and generates AI recommendations using GPT-4.
- **Nodes Involved:** Search Indoor Spots, Search Outdoor Spots, AI Generate Recommendations
- **Node Details:**

  - **Search Indoor Spots**
    - Type: HTTP Request node
    - Configuration: Calls Google Places Nearby Search API with keywords for indoor venues (e.g., "indoor shopping mall", "underground").
    - Inputs: Route by Weather Condition (Rain/Snow)
    - Outputs: AI Generate Recommendations
    - Edge cases: API errors, empty results.

  - **Search Outdoor Spots**
    - Type: HTTP Request node
    - Configuration: Calls Google Places Nearby Search API with keywords for outdoor venues (e.g., "terrace", "open air").
    - Inputs: Route by Weather Condition (Sunny/Cloudy)
    - Outputs: AI Generate Recommendations
    - Edge cases: API errors, empty results.

  - **AI Generate Recommendations**
    - Type: OpenAI (Langchain) node
    - Configuration:
      - Model: GPT-4.1-mini
      - Prompt: Combines user context, preferences, weather, available time, and candidate spots to generate top 3 friendly recommendations suitable for Slack.
    - Inputs: Search Indoor Spots or Search Outdoor Spots
    - Outputs: Send Slack Notification
    - Edge cases: API quota, prompt errors, malformed outputs.

#### 1.7 Slack Notification

- **Overview:** Sends the AI-generated recommendation message to the configured Slack channel.
- **Nodes Involved:** Send Slack Notification
- **Node Details:**

  - **Send Slack Notification**
    - Type: Slack node
    - Configuration:
      - Sends text message from AI Generate Recommendations output
      - Uses OAuth2 authentication
      - Requires Slack channel ID configuration
    - Inputs: AI Generate Recommendations
    - Outputs: None (end node)
    - Edge cases: Invalid Slack token or channel ID, network issues.

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                        | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                      |
|------------------------------|--------------------------------|-------------------------------------|--------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger             | scheduleTrigger                | Triggers workflow every 30 minutes  | None                           | Set Configuration                 |                                                                                                |
| Set Configuration           | set                           | Defines config variables and API keys | Schedule Trigger               | Get User Preferences from Notion  | ⚙️ Step 1: Configuration                                                                        |
| Get User Preferences from Notion | notion                       | Fetches user preferences             | Set Configuration              | Get Next Calendar Event           | 📅 Step 2: Fetch Calendar & Preferences                                                        |
| Get Next Calendar Event      | googleCalendar                | Gets next calendar event             | Get User Preferences from Notion | Get Weather at Destination        | 📅 Step 2: Fetch Calendar & Preferences                                                        |
| Get Weather at Destination   | openWeatherMap                | Retrieves current weather            | Get Next Calendar Event        | Get Travel Time via Google Maps   | 🌤️ Step 3: Weather & Travel Time                                                              |
| Get Travel Time via Google Maps | httpRequest                  | Calculates travel time to event      | Get Weather at Destination     | Calculate Available Gap Time      | 🌤️ Step 3: Weather & Travel Time                                                              |
| Calculate Available Gap Time | code                          | Computes gap time and validates      | Get Travel Time via Google Maps | Has Sufficient Gap Time?          | ⏱️ Step 4: Gap Time Calculation                                                                |
| Has Sufficient Gap Time?     | if                            | Checks if gap time meets threshold   | Calculate Available Gap Time   | Merge All Context Data            | ⏱️ Step 4: Gap Time Calculation                                                                |
| Merge All Context Data       | code                          | Combines all relevant context data   | Has Sufficient Gap Time?       | Route by Weather Condition        |                                                                                                |
| Route by Weather Condition   | switch                        | Routes flow based on weather ID      | Merge All Context Data         | Search Indoor Spots, Search Outdoor Spots | 🌧️☀️ Step 5: Weather-Based Routing                                                          |
| Search Indoor Spots          | httpRequest                   | Searches for indoor spots near event | Route by Weather Condition     | AI Generate Recommendations       |                                                                                                |
| Search Outdoor Spots         | httpRequest                   | Searches for outdoor spots near event | Route by Weather Condition     | AI Generate Recommendations       |                                                                                                |
| AI Generate Recommendations  | openAi (Langchain)            | Generates top 3 recommendations using GPT-4 | Search Indoor Spots, Search Outdoor Spots | Send Slack Notification          | 🤖 Step 6: AI Recommendation                                                                   |
| Send Slack Notification      | slack                         | Sends recommendation message to Slack | AI Generate Recommendations    | None                            | 📱 Step 7: Slack Notification                                                                  |
| Sticky Note                 | stickyNote                    | Various explanatory notes            | None                          | None                            | ☕ Gap Time Concierge - Smart Break Spot Recommender (covers entire workflow)                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**
   - Type: scheduleTrigger
   - Set to run every 30 minutes interval.

2. **Create a Set Node named "Set Configuration"**
   - Assign the following variables:
     - `currentLocation`: String, e.g. "Tokyo Station"
     - `googleMapsApiKey`: String, your Google Maps API key
     - `googlePlacesApiKey`: String, your Google Places API key
     - `openWeatherApiKey`: String, your OpenWeatherMap API key
     - `notionDatabaseId`: String, your Notion database ID for user preferences
     - `minGapTimeMinutes`: Number, default 30
   - Connect Schedule Trigger output to this node.

3. **Create a Notion Node named "Get User Preferences from Notion"**
   - Resource: databasePage
   - Operation: getAll
   - Database ID: Use expression to get from `Set Configuration` node JSON field `notionDatabaseId`.
   - Connect output of "Set Configuration" to this node.

4. **Create a Google Calendar Node named "Get Next Calendar Event"**
   - Operation: getAll
   - Limit: 1
   - Order by startTime, expand recurring events
   - TimeMin: Current time (expression: `new Date().toISOString()`)
   - Connect output of Notion node to this node.

5. **Create an OpenWeatherMap Node named "Get Weather at Destination"**
   - City Name: Use expression to get from calendar event location.
   - Connect output of calendar node to this node.

6. **Create an HTTP Request Node named "Get Travel Time via Google Maps"**
   - URL: Compose with expression including origin (currentLocation), destination (calendar location), mode=transit, and Google Maps API key.
   - Connect output of weather node to this node.

7. **Create a Code Node named "Calculate Available Gap Time"**
   - Paste JavaScript code that:
     - Extracts event start time, travel duration from Google Maps response
     - Calculates gap time (available minutes minus travel time)
     - Checks if gap time ≥ minGapTimeMinutes
   - Connect output of travel time node to this node.

8. **Create an If Node named "Has Sufficient Gap Time?"**
   - Condition: Check if `hasGapTime` (boolean) is true.
   - Connect output of the code node to this node.

9. **Create a Code Node named "Merge All Context Data"**
   - Merge gap time data, weather, calendar event, user preferences, and config into one JSON object.
   - Extract weather ID for routing.
   - Connect the "true" output of the If node to this node.

10. **Create a Switch Node named "Route by Weather Condition"**
    - Condition rules:
      - If weatherId < 700 → output "Rain/Snow"
      - Else → output "Sunny/Cloudy"
    - Connect output of Merge node here.

11. **Create two HTTP Request Nodes:**
    - "Search Indoor Spots"
      - Query Google Places Nearby Search API for indoor keywords around event location.
      - Use googlePlacesApiKey from config.
      - Connect "Rain/Snow" output of Switch node here.
    - "Search Outdoor Spots"
      - Query Google Places Nearby Search API for outdoor keywords similarly.
      - Connect "Sunny/Cloudy" output of Switch node here.

12. **Create an OpenAI Node named "AI Generate Recommendations"**
    - Model: GPT-4.1-mini
    - Prompt: Construct using merged context data and place search results.
    - Output plain text suitable for Slack.
    - Connect outputs of both spot search nodes to this node.

13. **Create a Slack Node named "Send Slack Notification"**
    - Authentication: OAuth2 with Slack
    - Channel: Configure your Slack channel ID
    - Message: Use AI node’s plain text output.
    - Connect output of AI Generate Recommendations here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is branded as "Gap Time Concierge - Smart Break Spot Recommender" to reflect its purpose and user benefits.                                                                | Workflow Sticky Note                                                                                |
| Setup requires integrating multiple APIs and OAuth connections: Google Calendar, Google Maps & Places, OpenWeatherMap, Notion, Slack, and OpenAI.                                       | Setup Requirements in Sticky Note                                                                  |
| The workflow runs every 30 minutes to keep recommendations timely and relevant.                                                                                                          | Schedule Trigger Sticky Note                                                                        |
| For best results, ensure that all API keys have necessary permissions and quota, and that the Notion database schema matches expected user preference data.                             | General best practices                                                                              |
| GPT-4 prompt is designed to output human-friendly Slack messages, not raw JSON, to simplify notification formatting.                                                                    | AI Generate Recommendations node details                                                          |
| Slack OAuth2 credentials must be configured with 'chat:write' scope to send messages to channels.                                                                                        | Slack node configuration                                                                            |
| Google Maps Directions API mode is set to 'transit' but can be adjusted to other modes (driving, walking) based on user needs.                                                          | Get Travel Time via Google Maps node configuration                                                |
| Weather condition routing uses OpenWeatherMap weather condition ID conventions, where IDs below 700 represent precipitation (rain/snow).                                                | Route by Weather Condition node details                                                           |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.