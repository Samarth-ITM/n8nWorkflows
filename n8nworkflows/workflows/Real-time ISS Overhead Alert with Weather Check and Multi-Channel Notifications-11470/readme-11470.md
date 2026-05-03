Real-time ISS Overhead Alert with Weather Check and Multi-Channel Notifications

https://n8nworkflows.xyz/workflows/real-time-iss-overhead-alert-with-weather-check-and-multi-channel-notifications-11470


# Real-time ISS Overhead Alert with Weather Check and Multi-Channel Notifications

### 1. Workflow Overview

This workflow, titled **"Real-time ISS Overhead Alert with Weather Check and Multi-Channel Notifications"**, is a comprehensive satellite tracking and alert system primarily focused on the International Space Station (ISS) and the Chinese space station Tiangong. It integrates real-time tracking, weather checks, AI-generated trivia, daily visibility predictions, weekly analytics, and multi-channel notifications. The workflow targets space enthusiasts and observers who want timely, detailed, and contextual alerts about satellite passes over their location, enriched with weather conditions and observational tips.

The workflow is logically divided into three main functional blocks:

- **1.1 Real-time Tracking and Notification (Triggered every 5 minutes)**
  - Fetch satellite positions in real-time.
  - Calculate proximity and observation conditions.
  - Query weather and astronaut data.
  - Generate an AI-powered trivia fact.
  - Format and dispatch rich notifications via Discord, Telegram, and Slack.
  - Log sightings to Google Sheets for historical tracking.

- **1.2 Daily Satellite Pass Predictions (Triggered daily at 6 AM)**
  - Retrieve 7-day pass predictions for ISS and Tiangong.
  - Process, rank, and format predictions into a detailed report in Japanese.
  - Notify users via Telegram and Discord.
  - Add best passes as events to Google Calendar.

- **1.3 Weekly Analytics and Reporting (Triggered Sundays at 8 PM)**
  - Load historical sightings from Google Sheets.
  - Compute weekly statistics and success metrics.
  - Use AI to generate an engaging weekly summary report in Japanese.
  - Send the report to Telegram and Discord channels.

Each block is supported with explanatory sticky notes and has clear node-to-node dependencies for modular operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Real-time Tracking and Notification (Every 5 minutes)

- **Overview:**  
  This block runs every 5 minutes to fetch current satellite positions, calculate observational parameters, check weather and astronaut presence, generate AI trivia, format a rich notification message, log data, and send multi-channel alerts.

- **Nodes Involved:**  
  - Real-time Check (5 min) [Schedule Trigger]  
  - Get ISS Position1 [HTTP Request]  
  - Get ISS Detailed (N2YO) [HTTP Request]  
  - Get Tiangong Position [HTTP Request]  
  - Calculate All Satellites [Code]  
  - Any Satellite Nearby? [If]  
  - Get Weather Conditions [HTTP Request]  
  - Get Astronauts in Space [HTTP Request]  
  - Analyze Observation Conditions1 [Code]  
  - Observable Conditions? [If]  
  - Generate Space Trivia [Langchain Chain LLM]  
  - OpenAI Trivia [Langchain OpenAI]  
  - Format Rich Notification [Code]  
  - Log to History [Google Sheets]  
  - Send to Discord2 [Discord]  
  - Send to Telegram3 [Telegram]  
  - Send to Slack1 [Slack]

- **Node Details:**

  - **Real-time Check (5 min)**  
    - Type: Schedule Trigger  
    - Role: Initiates the real-time flow every 5 minutes.  
    - Config: Triggers every 5 minutes interval.  
    - Output: Triggers "Get ISS Position1".

  - **Get ISS Position1**  
    - Type: HTTP Request  
    - Role: Fetches the current basic position of the ISS from Open Notify API.  
    - Config: URL `http://api.open-notify.org/iss-now.json`. No authentication.  
    - Output connections: Triggers "Get ISS Detailed (N2YO)" and "Get Tiangong Position".

  - **Get ISS Detailed (N2YO)**  
    - Type: HTTP Request  
    - Role: Fetches detailed ISS position and velocity from N2YO API using user location variables.  
    - Config: URL dynamically constructed with environment variables for latitude, longitude, and API key.  
    - Potential Errors: API key invalid or quota exceeded, network timeouts.

  - **Get Tiangong Position**  
    - Type: HTTP Request  
    - Role: Fetches Tiangong satellite position from N2YO API similar to ISS detailed.  
    - Config: URL uses Tiangong NORAD ID (48274) and user location variables.  
    - Potential Errors: Same as ISS detailed.

  - **Calculate All Satellites**  
    - Type: Code (JavaScript)  
    - Role: Calculates distance, bearing, elevation for ISS and Tiangong relative to user location using Haversine formula and trigonometric calculations.  
    - Key logic:  
      - Reads satellite data from previous nodes.  
      - Computes distance and direction including Japanese compass directions.  
      - Determines if satellites are within configurable visibility radius (default 800km).  
    - Inputs: Outputs from ISS basic, ISS detailed, and Tiangong position nodes.  
    - Outputs: JSON with satellites array, nearbySatellites filtered list, and metadata like timestamps.  
    - Edge cases: Missing or malformed satellite data can cause calculation errors; defaults are provided.

  - **Any Satellite Nearby?**  
    - Type: If  
    - Role: Checks if any satellites are within visibility radius (boolean flag from previous code node).  
    - If true: proceeds to get weather and astronaut data.  
    - Else: flow ends here for this cycle.

  - **Get Weather Conditions**  
    - Type: HTTP Request  
    - Role: Fetches weather data for user location from OpenWeatherMap API using API key credential.  
    - Config: URL dynamically constructed with latitude, longitude, and API key. Uses OpenWeatherMap credential type.  
    - Outputs weather info with temperature, cloudiness, humidity, etc.  
    - Failure modes: API limits, invalid credentials, network errors.

  - **Get Astronauts in Space**  
    - Type: HTTP Request  
    - Role: Retrieves current humans in space from Open Notify API.  
    - Config: URL `http://api.open-notify.org/astros.json`. No auth.  
    - Outputs: List of astronauts and crafts.

  - **Analyze Observation Conditions1**  
    - Type: Code (JavaScript)  
    - Role: Combines satellite, weather, and astronaut data to compute observation score (0-100), time conditions, cloudiness impact, and photography settings recommendations.  
    - Logic highlights:  
       - Deducts points for bad weather (rain, snow, thunderstorm).  
       - Scores cloudiness levels.  
       - Evaluates time of day for twilight benefits.  
       - Prepares camera ISO, aperture, shutter speed tips based on score and time.  
    - Outputs consolidated observation context including astronaut details.  
    - Edge cases: Missing weather or astronaut data handled gracefully.

  - **Observable Conditions?**  
    - Type: If  
    - Role: Checks if observation score meets threshold (≥40) to proceed with trivia generation.  
    - Else: flow could end or skip trivia generation.

  - **Generate Space Trivia**  
    - Type: Langchain Chain LLM  
    - Role: Generates a Japanese space trivia fact based on current satellite proximity, astronaut count, and weather context.  
    - Model: GPT-4o-mini, temperature 0.8, max 300 tokens.  
    - Output: JSON with trivia in Japanese and English, category, emoji.  
    - Edge cases: AI response parsing may fail; fallback trivia is provided downstream.

  - **OpenAI Trivia**  
    - Type: Langchain OpenAI Node  
    - Role: Executes the LLM call for trivia generation (invoked by the chain node above).  
    - Requires OpenAI credentials.

  - **Format Rich Notification**  
    - Type: Code (JavaScript)  
    - Role: Builds a rich markdown message including satellite details, observation scores, photography tips, astronauts currently in space, and the AI-generated trivia.  
    - Also includes a Google Earth 3D map link for the satellite position.  
    - Outputs a detailed notificationMessage and a shortMessage for concise alerts.  
    - Handles JSON parsing failures of the trivia gracefully with fallback content.

  - **Log to History**  
    - Type: Google Sheets  
    - Role: Appends the observation event data (weather, satellite, observation score, trivia category, etc.) to a Google Sheet for historical tracking.  
    - Requires Google Sheets credentials and sheet/document ID environment variables.

  - **Send to Discord2**, **Send to Telegram3**, **Send to Slack1**  
    - Type: Discord, Telegram, Slack nodes  
    - Role: Send the formatted rich notification or short message to respective channels using webhook or bot credentials.  
    - Telegram and Discord use markdown formatting.  
    - Slack uses simple text message.

---

#### 2.2 Daily Satellite Pass Predictions (Daily at 6 AM)

- **Overview:**  
  This block fetches 7-day satellite pass predictions for ISS and Tiangong, processes and ranks these passes, formats a detailed Japanese report, sends notifications, and adds calendar events for best observation opportunities.

- **Nodes Involved:**  
  - Daily Predictions (6 AM) [Schedule Trigger]  
  - Get ISS Pass Predictions [HTTP Request]  
  - Get Tiangong Predictions [HTTP Request]  
  - Process All Predictions [Code]  
  - Format Prediction Report [Code]  
  - Send Predictions to Telegram [Telegram]  
  - Send Predictions to Discord [Discord]  
  - Loop Calendar Events [SplitInBatches]  
  - Add to Google Calendar [Google Calendar]

- **Node Details:**

  - **Daily Predictions (6 AM)**  
    - Type: Schedule Trigger  
    - Role: Fires daily at 6 AM to initiate prediction fetching.  
    - Output: Triggers both ISS and Tiangong pass prediction requests.

  - **Get ISS Pass Predictions**  
    - Type: HTTP Request  
    - Role: Queries N2YO API for ISS visual passes over the next 7 days from user location.  
    - Uses NORAD ID 25544 and environment variables for coordinates and API key.

  - **Get Tiangong Predictions**  
    - Type: HTTP Request  
    - Role: Same as above but for Tiangong satellite (NORAD ID 48274).

  - **Process All Predictions**  
    - Type: Code (JavaScript)  
    - Role: Combines pass data from ISS and Tiangong, formats each pass with times, durations, elevations, azimuths, compass directions (Japanese), and brightness estimation based on magnitude.  
    - Sorts passes chronologically.  
    - Extracts top 5 best passes (max elevation ≥40°).  
    - Outputs structured data with predictions and bestPasses arrays.

  - **Format Prediction Report**  
    - Type: Code (JavaScript)  
    - Role: Generates a Japanese markdown report summarizing:  
      - Best observation opportunities with detailed info.  
      - Full 7-day pass schedule (first 10 shown).  
      - Observation tips for satellite viewing.  
    - Includes Japanese characters and emoji for readability.

  - **Send Predictions to Telegram**  
    - Type: Telegram  
    - Role: Sends the formatted prediction message to users via Telegram chat using a configured chat ID.  
    - Uses Markdown formatting.

  - **Send Predictions to Discord**  
    - Type: Discord  
    - Role: Sends the prediction report to Discord via webhook.

  - **Loop Calendar Events**  
    - Type: SplitInBatches  
    - Role: Iterates over best passes to add them individually to Google Calendar.

  - **Add to Google Calendar**  
    - Type: Google Calendar  
    - Role: Creates calendar events with pass details (title, start/end time, description).  
    - Uses primary calendar.

---

#### 2.3 Weekly Analytics and Reporting (Sunday 8 PM)

- **Overview:**  
  This block aggregates historical sighting data weekly, computes statistics and breakdowns, generates an AI-powered analytical report in Japanese, and sends it to Telegram and Discord.

- **Nodes Involved:**  
  - Weekly Report (Sunday 8 PM) [Schedule Trigger]  
  - Get History Data [Google Sheets]  
  - Calculate Weekly Statistics [Code]  
  - Generate Weekly Report [Langchain Chain LLM]  
  - OpenAI Report [Langchain OpenAI]  
  - Format Weekly Report [Code]  
  - Send Weekly to Telegram [Telegram]  
  - Send Weekly to Discord [Discord]

- **Node Details:**

  - **Weekly Report (Sunday 8 PM)**  
    - Type: Schedule Trigger  
    - Role: Triggers the weekly analytics flow every Sunday at 8 PM.

  - **Get History Data**  
    - Type: Google Sheets  
    - Role: Retrieves all sighting records from the configured Google Sheet.

  - **Calculate Weekly Statistics**  
    - Type: Code (JavaScript)  
    - Role: Filters data for the last 7 days, computes:  
      - Total sightings, observable sightings, success rates.  
      - Average observation score.  
      - Breakdown counts by satellite, viewing direction, weather, and trivia categories.  
      - Identifies the best sighting by score.

  - **Generate Weekly Report**  
    - Type: Langchain Chain LLM  
    - Role: Uses GPT-4o-mini to generate a detailed weekly report in Japanese covering:  
      - Weekly highlights.  
      - Success rate analysis.  
      - Outlook for the following week.  
      - Advice for observers.

  - **OpenAI Report**  
    - Type: Langchain OpenAI Node  
    - Role: Executes the AI model call for the weekly report generation.

  - **Format Weekly Report**  
    - Type: Code (JavaScript)  
    - Role: Combines statistical data and AI output into a nicely formatted markdown message.  
    - Adds section headers, summary stats, best pass highlight, and AI-generated insights.

  - **Send Weekly to Telegram**  
    - Type: Telegram  
    - Role: Sends the weekly report message to configured Telegram chat.

  - **Send Weekly to Discord**  
    - Type: Discord  
    - Role: Sends the weekly report to Discord via webhook.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                            | Input Node(s)                        | Output Node(s)                                     | Sticky Note                                                                                                                       |
|------------------------------|----------------------------------|--------------------------------------------|------------------------------------|---------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Real-time Check (5 min)       | Schedule Trigger                 | Initiates real-time satellite tracking     | -                                  | Get ISS Position1                                  | 🔄 Real-time Tracking Flow: Every 5 minutes: Check satellite positions → Weather check → AI trivia → Log & Notify               |
| Get ISS Position1             | HTTP Request                    | Fetch ISS basic position                    | Real-time Check (5 min)             | Get ISS Detailed (N2YO), Get Tiangong Position     | Step 1 - Fetch Positions: Get current positions from Open Notify API and N2YO API                                                |
| Get ISS Detailed (N2YO)       | HTTP Request                    | Fetch detailed ISS position and velocity   | Get ISS Position1                   | Calculate All Satellites                           | Step 1 - Fetch Positions                                                                                                         |
| Get Tiangong Position         | HTTP Request                    | Fetch Tiangong position                     | Get ISS Position1                   | Calculate All Satellites                           | Step 1 - Fetch Positions                                                                                                         |
| Calculate All Satellites      | Code                           | Calculate distance, bearing, elevation     | Get ISS Detailed, Get Tiangong Position | Any Satellite Nearby?                            | Step 2 - Calculate: Distance & direction calculations                                                                           |
| Any Satellite Nearby?         | If                             | Check if satellites are within visibility  | Calculate All Satellites            | Get Weather Conditions, Get Astronauts in Space  | Step 3 - Filter Nearby: Filter satellites within visibility radius                                                             |
| Get Weather Conditions        | HTTP Request                   | Fetch local weather data                     | Any Satellite Nearby?               | Analyze Observation Conditions1                    | Step 4 - Context: Weather conditions                                                                                            |
| Get Astronauts in Space       | HTTP Request                   | Fetch current astronauts in space           | Any Satellite Nearby?               | Analyze Observation Conditions1                    | Step 4 - Context: Astronaut data                                                                                               |
| Analyze Observation Conditions1 | Code                        | Compute observation score and conditions    | Get Weather Conditions, Get Astronauts in Space, Calculate All Satellites | Observable Conditions?                     | Step 5 - Score: Weather, time, and photography scoring                                                                         |
| Observable Conditions?        | If                             | Check if observation score permits trivia   | Analyze Observation Conditions1    | Generate Space Trivia                              | Step 6 - AI Trivia: Generate AI trivia if observation conditions are good                                                      |
| Generate Space Trivia         | Langchain Chain LLM            | Generate Japanese space trivia fact         | Observable Conditions?              | Format Rich Notification                           | Step 6 - AI Trivia                                                                                                               |
| OpenAI Trivia                | Langchain OpenAI               | Invoke OpenAI model for trivia generation   | Generate Space Trivia               | Format Rich Notification                           | Step 6 - AI Trivia                                                                                                               |
| Format Rich Notification      | Code                           | Build rich notification message              | Generate Space Trivia               | Log to History                                    | Step 7 - Format: Rich message with satellite details, trivia, and observation tips                                              |
| Log to History                | Google Sheets                  | Log observation and weather data             | Format Rich Notification           | Send to Discord2, Send to Telegram3, Send to Slack1 | Step 8 - Log & Send: Save sightings and notify all channels                                                                    |
| Send to Discord2              | Discord                       | Send notification to Discord channel         | Log to History                    | -                                                 | Step 8 - Log & Send                                                                                                              |
| Send to Telegram3             | Telegram                      | Send notification to Telegram chat           | Log to History                    | -                                                 | Step 8 - Log & Send                                                                                                              |
| Send to Slack1                | Slack                         | Send short notification to Slack channel     | Log to History                    | -                                                 | Step 8 - Log & Send                                                                                                              |
| Daily Predictions (6 AM)      | Schedule Trigger              | Trigger daily pass predictions fetch         | -                                  | Get ISS Pass Predictions, Get Tiangong Predictions | 🔮 Daily Predictions Flow: Every morning at 6 AM: Fetch 7-day predictions → Format report → Add to Google Calendar               |
| Get ISS Pass Predictions      | HTTP Request                 | Fetch 7-day ISS pass predictions              | Daily Predictions (6 AM)            | Process All Predictions                            | Step 1 - Fetch Predictions                                                                                                       |
| Get Tiangong Predictions      | HTTP Request                 | Fetch 7-day Tiangong pass predictions         | Daily Predictions (6 AM)            | Process All Predictions                            | Step 1 - Fetch Predictions                                                                                                       |
| Process All Predictions       | Code                         | Combine, sort, rank pass predictions          | Get ISS Pass Predictions, Get Tiangong Predictions | Format Prediction Report                       | Step 2 - Process: Combine and rank passes                                                                                       |
| Format Prediction Report      | Code                         | Format Japanese report including tips          | Process All Predictions            | Send Predictions to Telegram, Send Predictions to Discord, Loop Calendar Events | Step 3 - Format Report                                                                                                           |
| Send Predictions to Telegram  | Telegram                    | Send prediction report to Telegram             | Format Prediction Report          | -                                                 | Step 4 - Notify Users                                                                                                            |
| Send Predictions to Discord   | Discord                     | Send prediction report to Discord              | Format Prediction Report          | -                                                 | Step 4 - Notify Users                                                                                                            |
| Loop Calendar Events          | SplitInBatches              | Iterate best passes to add to calendar          | Format Prediction Report          | Add to Google Calendar                            | Step 5 - Add to Calendar                                                                                                         |
| Add to Google Calendar        | Google Calendar             | Add best satellite passes as calendar events    | Loop Calendar Events              | Loop Calendar Events (loop back)                  | Step 5 - Add to Calendar                                                                                                         |
| Weekly Report (Sunday 8 PM)   | Schedule Trigger             | Trigger weekly analytics and reporting          | -                              | Get History Data                                  | 📊 Weekly Analytics Flow: Aggregate history → AI analysis → Generate report                                                    |
| Get History Data              | Google Sheets               | Load historical sightings data                   | Weekly Report (Sunday 8 PM)       | Calculate Weekly Statistics                        | Step 1 - Load History                                                                                                            |
| Calculate Weekly Statistics   | Code                        | Compute weekly metrics and breakdowns             | Get History Data                 | Generate Weekly Report                            | Step 2 - Stats                                                                                                                   |
| Generate Weekly Report        | Langchain Chain LLM         | Generate AI weekly analysis report                | Calculate Weekly Statistics       | Format Weekly Report                              | Step 3 - AI Analysis                                                                                                             |
| OpenAI Report                | Langchain OpenAI            | Execute AI model for weekly report                 | Generate Weekly Report            | Format Weekly Report                              | Step 3 - AI Analysis                                                                                                             |
| Format Weekly Report          | Code                        | Combine stats and AI report into final message      | Generate Weekly Report, Calculate Weekly Statistics | Send Weekly to Telegram, Send Weekly to Discord | Step 4 - Send Report                                                                                                             |
| Send Weekly to Telegram       | Telegram                    | Send weekly report to Telegram                      | Format Weekly Report             | -                                                 | Step 4 - Send Report                                                                                                             |
| Send Weekly to Discord        | Discord                     | Send weekly report to Discord                       | Format Weekly Report             | -                                                 | Step 4 - Send Report                                                                                                             |
| Workflow Description1         | Sticky Note                 | Describes entire workflow purpose and requirements | -                              | -                                                 | General workflow overview and required services                                                                                |
| Real-time Note                | Sticky Note                 | Notes real-time tracking flow steps                 | -                              | -                                                 | 🔄 Real-time Tracking Flow                                                                                                       |
| Prediction Note               | Sticky Note                 | Notes daily predictions flow steps                   | -                              | -                                                 | 🔮 Daily Predictions Flow                                                                                                        |
| Weekly Note                  | Sticky Note                 | Notes weekly analytics flow steps                    | -                              | -                                                 | 📊 Weekly Analytics Flow                                                                                                        |
| Step 1 - Fetch Positions      | Sticky Note                 | Describes initial data fetching step                  | -                              | -                                                 | Step 1 - Fetch Positions                                                                                                        |
| Step 2 - Calculate            | Sticky Note                 | Describes calculation of distances and bearings       | -                              | -                                                 | Step 2 - Calculate                                                                                                              |
| Step 3 - Filter Nearby        | Sticky Note                 | Describes filtering satellites by proximity            | -                              | -                                                 | Step 3 - Check Proximity                                                                                                        |
| Step 4 - Context              | Sticky Note                 | Describes fetching weather and astronaut data          | -                              | -                                                 | Step 4 - Context                                                                                                                |
| Step 5 - Score                | Sticky Note                 | Describes scoring observation conditions                | -                              | -                                                 | Step 5 - Score                                                                                                                  |
| Step 6 - AI Trivia            | Sticky Note                 | Describes AI trivia generation                            | -                              | -                                                 | Step 6 - AI Trivia                                                                                                              |
| Step 7 - Format               | Sticky Note                 | Describes formatting notification                          | -                              | -                                                 | Step 7 - Format                                                                                                                 |
| Step 8 - Log & Send           | Sticky Note                 | Describes logging and notification dispatch               | -                              | -                                                 | Step 8 - Log & Notify                                                                                                           |
| Step 1 - Fetch Predictions    | Sticky Note                 | Describes fetching satellite pass predictions             | -                              | -                                                 | Step 1 - Fetch Predictions                                                                                                     |
| Step 2 - Process              | Sticky Note                 | Describes processing and ranking predictions               | -                              | -                                                 | Step 2 - Process                                                                                                               |
| Step 3 - Format Report        | Sticky Note                 | Describes formatting prediction report                      | -                              | -                                                 | Step 3 - Format Report                                                                                                         |
| Step 4 - Notify               | Sticky Note                 | Describes sending prediction notifications                   | -                              | -                                                 | Step 4 - Notify Users                                                                                                          |
| Step 5 - Calendar             | Sticky Note                 | Describes adding prediction events to Google Calendar         | -                              | -                                                 | Step 5 - Add to Calendar                                                                                                       |
| Step 1 - Load History         | Sticky Note                 | Describes loading historical sightings data                  | -                              | -                                                 | Step 1 - Load History                                                                                                          |
| Step 2 - Stats               | Sticky Note                 | Describes calculating weekly statistics                       | -                              | -                                                 | Step 2 - Calculate Statistics                                                                                                |
| Step 3 - AI Analysis          | Sticky Note                 | Describes AI analysis for the weekly report                    | -                              | -                                                 | Step 3 - AI Analysis                                                                                                          |
| Step 4 - Send Report          | Sticky Note                 | Describes formatting and sending weekly report                  | -                              | -                                                 | Step 4 - Format & Send Report                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Environment Variables:**
   - USER_LAT (e.g., 35.6762)
   - USER_LON (e.g., 139.6503)
   - USER_LOCATION_NAME (e.g., "Tokyo")
   - VISIBILITY_RADIUS_KM (default 800)
   - N2YO_API_KEY
   - GOOGLE_SHEET_ID
   - TELEGRAM_CHAT_ID
   - Slack and Discord webhook URLs as needed

---

#### Real-time Tracking and Notification

2. **Add Schedule Trigger "Real-time Check (5 min)"**
   - Trigger interval: Every 5 minutes.

3. **Add HTTP Request "Get ISS Position1"**
   - URL: `http://api.open-notify.org/iss-now.json`
   - Connect "Real-time Check (5 min)" → "Get ISS Position1"

4. **Add HTTP Request "Get ISS Detailed (N2YO)"**
   - URL: `https://api.n2yo.com/rest/v1/satellite/positions/25544/{{ $vars.USER_LAT }}/{{ $vars.USER_LON }}/0/1/?apiKey={{ $vars.N2YO_API_KEY }}`
   - Connect "Get ISS Position1" → "Get ISS Detailed (N2YO)"

5. **Add HTTP Request "Get Tiangong Position"**
   - URL: `https://api.n2yo.com/rest/v1/satellite/positions/48274/{{ $vars.USER_LAT }}/{{ $vars.USER_LON }}/0/1/?apiKey={{ $vars.N2YO_API_KEY }}`
   - Connect "Get ISS Position1" → "Get Tiangong Position"

6. **Add Code Node "Calculate All Satellites"**
   - Paste the provided JavaScript code that calculates distance, bearing, elevation, direction in English and Japanese, and filters satellites within visibility radius.
   - Connect both "Get ISS Detailed (N2YO)" and "Get Tiangong Position" → "Calculate All Satellites"

7. **Add If Node "Any Satellite Nearby?"**
   - Condition: `$json.hasNearby` is boolean true.
   - Connect "Calculate All Satellites" → "Any Satellite Nearby?"

8. **Add HTTP Request "Get Weather Conditions"**
   - URL: `https://api.openweathermap.org/data/2.5/weather?lat={{ $json.userLocation.latitude }}&lon={{ $json.userLocation.longitude }}&appid={{ $credentials.openWeatherMapApi.apiKey }}&units=metric`
   - Authentication: Use OpenWeatherMap API credential.
   - Connect "Any Satellite Nearby?" (true) → "Get Weather Conditions"

9. **Add HTTP Request "Get Astronauts in Space"**
   - URL: `http://api.open-notify.org/astros.json`
   - Connect "Any Satellite Nearby?" (true) → "Get Astronauts in Space"

10. **Add Code Node "Analyze Observation Conditions1"**
    - Combine satellite data, weather, and astronaut info.
    - Calculate observation score, conditions, time of day effects, and photography tips.
    - Connect both "Get Weather Conditions" and "Get Astronauts in Space" → "Analyze Observation Conditions1"

11. **Add If Node "Observable Conditions?"**
    - Condition: `$json.observation.canObserve` is boolean true.
    - Connect "Analyze Observation Conditions1" → "Observable Conditions?"

12. **Add Langchain Chain LLM Node "Generate Space Trivia"**
    - Model: GPT-4o-mini  
    - Temperature: 0.8  
    - Max Tokens: 300  
    - Prompt: Japanese trivia about space topics with current context variables.  
    - Connect "Observable Conditions?" (true) → "Generate Space Trivia"

13. **Add Langchain OpenAI Node "OpenAI Trivia"**
    - Connect "Generate Space Trivia" → "OpenAI Trivia"

14. **Add Code Node "Format Rich Notification"**
    - Build multi-section markdown message including satellite info, observations, trivia, astronaut list, and Google Earth link.  
    - Connect "OpenAI Trivia" → "Format Rich Notification"

15. **Add Google Sheets Node "Log to History"**
    - Append mode, map columns for weather, satellite, distance, observation score, trivia category, timestamp, etc.  
    - Connect "Format Rich Notification" → "Log to History"

16. **Add Discord Node "Send to Discord2"**
    - Use webhook, send the detailed notificationMessage.  
    - Connect "Log to History" → "Send to Discord2"

17. **Add Telegram Node "Send to Telegram3"**
    - Use chat ID variable, Markdown parse mode, send notificationMessage.  
    - Connect "Log to History" → "Send to Telegram3"

18. **Add Slack Node "Send to Slack1"**
    - Use webhook URL, send shortMessage.  
    - Connect "Log to History" → "Send to Slack1"

---

#### Daily Satellite Pass Predictions

19. **Add Schedule Trigger "Daily Predictions (6 AM)"**
    - Trigger at 6:00 AM every day.

20. **Add HTTP Request Nodes:**
    - "Get ISS Pass Predictions"  
      URL: `https://api.n2yo.com/rest/v1/satellite/visualpasses/25544/{{ $vars.USER_LAT }}/{{ $vars.USER_LON }}/0/7/300/?apiKey={{ $vars.N2YO_API_KEY }}`
    - "Get Tiangong Predictions"  
      URL: `https://api.n2yo.com/rest/v1/satellite/visualpasses/48274/{{ $vars.USER_LAT }}/{{ $vars.USER_LON }}/0/7/300/?apiKey={{ $vars.N2YO_API_KEY }}`  
    - Connect "Daily Predictions (6 AM)" → both prediction nodes.

21. **Add Code Node "Process All Predictions"**
    - Combine, format, rank passes, determine brightness, and filter best passes (elevation ≥40°).  
    - Connect both prediction nodes → "Process All Predictions"

22. **Add Code Node "Format Prediction Report"**
    - Create a Japanese markdown report with top passes, full schedule, and observation tips.  
    - Connect "Process All Predictions" → "Format Prediction Report"

23. **Add Telegram Node "Send Predictions to Telegram"**
    - Send formatted predictionMessage to Telegram chat.  
    - Connect "Format Prediction Report" → "Send Predictions to Telegram"

24. **Add Discord Node "Send Predictions to Discord"**
    - Send predictionMessage to Discord webhook.  
    - Connect "Format Prediction Report" → "Send Predictions to Discord"

25. **Add SplitInBatches Node "Loop Calendar Events"**
    - Loop over best passes to add calendar events.  
    - Connect "Format Prediction Report" → "Loop Calendar Events"

26. **Add Google Calendar Node "Add to Google Calendar"**
    - Create events with start/end times, title, and description.  
    - Connect "Loop Calendar Events" → "Add to Google Calendar"  
    - Connect "Add to Google Calendar" → "Loop Calendar Events" (loop back for batch processing)

---

#### Weekly Analytics and Reporting

27. **Add Schedule Trigger "Weekly Report (Sunday 8 PM)"**
    - Trigger weekly every Sunday at 20:00.

28. **Add Google Sheets Node "Get History Data"**
    - Read all rows from configured Google Sheet.  
    - Connect "Weekly Report (Sunday 8 PM)" → "Get History Data"

29. **Add Code Node "Calculate Weekly Statistics"**
    - Filter last 7 days, calculate counts, averages, breakdowns, best sightings.  
    - Connect "Get History Data" → "Calculate Weekly Statistics"

30. **Add Langchain Chain LLM Node "Generate Weekly Report"**
    - Use GPT-4o-mini to create a detailed Japanese weekly report with stats and insights.  
    - Connect "Calculate Weekly Statistics" → "Generate Weekly Report"

31. **Add Langchain OpenAI Node "OpenAI Report"**
    - Execute AI model call for report generation.  
    - Connect "Generate Weekly Report" → "OpenAI Report"

32. **Add Code Node "Format Weekly Report"**
    - Combine stats and AI report into a markdown message.  
    - Connect "OpenAI Report", "Calculate Weekly Statistics" → "Format Weekly Report"

33. **Add Telegram Node "Send Weekly to Telegram"**
    - Send weekly report message to Telegram chat.  
    - Connect "Format Weekly Report" → "Send Weekly to Telegram"

34. **Add Discord Node "Send Weekly to Discord"**
    - Send weekly report to Discord webhook.  
    - Connect "Format Weekly Report" → "Send Weekly to Discord"

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow tracks ISS and Tiangong satellites in real-time, provides weather-aware observation scoring, AI trivia, and weekly reports. | Workflow Description sticky note summarizing key features and required services.                               |
| Required services include N2YO API for satellite data, OpenWeatherMap for weather, OpenAI for AI generation, Google Sheets & Calendar for data storage and events, and Discord/Telegram/Slack for notifications. | Workflow Description sticky note.                                                                              |
| Observation scoring favors twilight hours and clear skies; photography settings are dynamically recommended based on conditions.  | Analyze Observation Conditions1 node comments.                                                                |
| Satellite visibility radius is configurable via environment variable VISIBILITY_RADIUS_KM (default 800 km).                       | Calculate All Satellites node and Step 3 - Filter Nearby sticky note.                                          |
| AI-generated trivia and weekly reports use GPT-4o-mini model via Langchain integration.                                            | Generate Space Trivia and Generate Weekly Report nodes.                                                        |
| Notifications support rich markdown messages on Telegram and Discord; Slack receives short text updates.                          | Format Rich Notification node and multi-channel send nodes.                                                   |
| Google Sheets schema includes timestamp, satellite, distance, direction, elevation, observation score, weather, and trivia category. | Log to History node and Workflow Description sticky note.                                                     |
| 3D satellite position links use Google Earth Web for visualization.                                                               | Format Rich Notification node includes link to `https://earth.google.com/web/search/{lat},{lon}`               |

---

This detailed structured reference enables comprehensive understanding, modification, and reproduction of the workflow by advanced users or AI agents.