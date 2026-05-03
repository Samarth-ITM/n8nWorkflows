Personalized Weather Assistant with Google Calendar, WeatherAPI, Gemini & Telegram

https://n8nworkflows.xyz/workflows/personalized-weather-assistant-with-google-calendar--weatherapi--gemini---telegram-6593


# Personalized Weather Assistant with Google Calendar, WeatherAPI, Gemini & Telegram

---
### 1. Workflow Overview

This workflow, **Personalized Weather Assistant with Google Calendar, WeatherAPI, AI & Telegram**, automates the delivery of a personalized daily agenda message. It combines scheduled retrieval of Google Calendar events for today and tomorrow with real-time local weather conditions fetched from WeatherAPI. An AI agent powered by Google Gemini via OpenRouter summarizes and formats the combined data into a friendly, motivating message. The final summary is sent to the user via Telegram.

**Target Use Cases:**  
- Professionals or individuals who want a concise overview of their upcoming events with local weather insights to better plan their day.  
- Automated daily notifications combining calendar and weather data in a human-friendly format.  

**Logical Blocks:**  
- **1.1 Scheduled Trigger & Timezone Setup:** Automatically triggers workflow at 6 AM daily and sets the timezone context for subsequent queries.  
- **1.2 Google Calendar Event Retrieval:** Fetches all calendar events for today and tomorrow from the user's primary Google Calendar.  
- **1.3 Event Location Check:** Filters events to distinguish those with and without location data.  
- **1.4 Weather Data Fetching:** For events with location, calls WeatherAPI to get current weather conditions at event time and place.  
- **1.5 AI Summarization with Weather:** Uses AI Agent with Google Gemini model to generate a personalized agenda message including weather details.  
- **1.6 AI Summarization without Weather:** For events without location, generates an agenda message without weather data.  
- **1.7 Telegram Notification:** Sends the AI-generated message to the user via Telegram bot.  

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Timezone Setup

**Overview:**  
This block initiates the workflow daily at 6 AM and sets the timezone for accurate event queries.

**Nodes Involved:**  
- Schedule Trigger  
- Set Timezone

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow daily at a fixed time (6 AM).  
  - *Configuration:* Triggers at hour 6 every day.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects to Set Timezone node.  
  - *Edge Cases:* If n8n instance is down or paused at trigger time, the workflow won’t start.  
  - *Version:* 1.2

- **Set Timezone**  
  - *Type:* Set  
  - *Role:* Assigns a default timezone ("UTC") to a variable `Timezone` for use in event queries.  
  - *Configuration:* Sets string variable `Timezone` = "UTC".  
  - *Inputs:* Triggered by Schedule Trigger output.  
  - *Outputs:* Passes data to Google Calendar node.  
  - *Edge Cases:* User should modify timezone value to their local timezone to ensure correct event times.  
  - *Version:* 3.4

---

#### 1.2 Google Calendar Event Retrieval

**Overview:**  
Retrieves all events scheduled from the current time (in the set timezone) up to 24 hours later from the user's primary Google Calendar.

**Nodes Involved:**  
- Get many events

**Node Details:**

- **Get many events**  
  - *Type:* Google Calendar  
  - *Role:* Fetches all calendar events for today and tomorrow.  
  - *Configuration:*  
    - Operation: Get all events  
    - Time range: From `$now` at `Timezone` to `$now + 1 day` at `Timezone`  
    - Calendar: Primary calendar of the connected Google account  
  - *Inputs:* Receives timezone data from Set Timezone node.  
  - *Outputs:* Sends event data downstream to the If node.  
  - *Expressions:* Uses expression to calculate timeMin and timeMax dynamically based on timezone.  
  - *Credentials:* Requires OAuth2 credentials for Google Calendar.  
  - *Edge Cases:*  
    - No events found will result in empty output.  
    - OAuth token expiration or permission issues can cause failure.  
  - *Version:* 1.3

---

#### 1.3 Event Location Check

**Overview:**  
Determines which events have location information to decide if weather data should be fetched.

**Nodes Involved:**  
- If

**Node Details:**

- **If**  
  - *Type:* If (conditional)  
  - *Role:* Checks if event item has a non-empty `location` field.  
  - *Configuration:* Condition tests if `location` field is not empty.  
  - *Inputs:* Receives all events from Get many events node.  
  - *Outputs:*  
    - True branch: events with location -> HTTP Request node for weather data.  
    - False branch: events without location -> AI Agent2 for agenda without weather.  
  - *Edge Cases:*  
    - Events with malformed or missing location fields may be misclassified.  
  - *Version:* 2.2

---

#### 1.4 Weather Data Fetching

**Overview:**  
For each event with location, fetches current weather data from WeatherAPI based on event location, date, and hour.

**Nodes Involved:**  
- HTTP Request

**Node Details:**

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Calls WeatherAPI to get weather details for event location and time.  
  - *Configuration:*  
    - URL: `http://api.weatherapi.com/v1/current.json`  
    - Query parameters:  
      - `key`: WeatherAPI key (user must replace placeholder)  
      - `q`: Event location  
      - `dt`: Event start date (parsed from ISO datetime)  
      - `hour`: Event start hour (parsed from ISO datetime)  
  - *Inputs:* Receives event with location from If node.  
  - *Outputs:* Sends weather data combined with event data to AI Agent node.  
  - *Edge Cases:*  
    - Invalid or missing WeatherAPI key causes authentication failure.  
    - Incorrect location string might yield no weather data.  
    - API rate limits or downtime can cause failures.  
  - *Version:* 4.2

---

#### 1.5 AI Summarization with Weather

**Overview:**  
Uses Langchain AI Agent with Google Gemini model to create a concise, friendly agenda message including weather details.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- Send a text message (Telegram)

**Node Details:**

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Formats agenda and weather into a personalized text summary.  
  - *Configuration:*  
    - Text prompt interpolates event agenda and weather parameters (temperature, humidity, UV index, wind, visibility).  
    - System message guides AI to be polite, concise, simple, and include emojis.  
    - Output parsed for clean text.  
  - *Inputs:* Receives event + weather data from HTTP Request.  
  - *Outputs:* Sends formatted message to Telegram node.  
  - *Edge Cases:*  
    - Expression errors if expected weather fields are missing.  
    - AI response latency or errors if OpenRouter service is unavailable.  
  - *Version:* 2

- **OpenRouter Chat Model**  
  - *Type:* Langchain OpenRouter Chat Model  
  - *Role:* Provides the language model (Google Gemini 2.0 Flash) for AI Agent.  
  - *Configuration:* Model set to `"google/gemini-2.0-flash-exp:free"`.  
  - *Inputs:* Connected as AI model resource for AI Agent.  
  - *Outputs:* Feeds AI Agent.  
  - *Credentials:* Requires OpenRouter API key in n8n.  
  - *Edge Cases:* Service downtime or credential issues.  
  - *Version:* 1

- **Send a text message (Telegram)**  
  - *Type:* Telegram node  
  - *Role:* Sends the AI-generated agenda message to user's Telegram chat.  
  - *Configuration:*  
    - Text field set to AI Agent output.  
    - chatId and webhookId must be user-specific.  
    - Attribution appended is disabled.  
  - *Inputs:* Receives message text from AI Agent.  
  - *Credentials:* Requires Telegram Bot token and chat_id setup.  
  - *Edge Cases:*  
    - Invalid chatId or bot token causes send failure.  
    - Telegram API limits or outages.  
  - *Version:* 1.2

---

#### 1.6 AI Summarization without Weather

**Overview:**  
For events without location, this block generates a simple agenda summary without weather data.

**Nodes Involved:**  
- AI Agent2  
- OpenRouter Chat Model2  
- Send a text message1 (Telegram)

**Node Details:**

- **AI Agent2**  
  - *Type:* Langchain Agent  
  - *Role:* Creates agenda message with event name and time, excluding weather.  
  - *Configuration:*  
    - Prompt includes event summary and start/end date only.  
    - System message instructs simple, polite text with emojis.  
  - *Inputs:* Events without location from If node false branch.  
  - *Outputs:* Passes formatted message to Telegram node.  
  - *Edge Cases:* Same as AI Agent, plus risk of missing start/end dates.  
  - *Version:* 2

- **OpenRouter Chat Model2**  
  - *Type:* Langchain OpenRouter Chat Model  
  - *Role:* Provides language model resource for AI Agent2.  
  - *Configuration:* Same Gemini 2.0 model.  
  - *Inputs:* Feeds AI Agent2.  
  - *Credentials:* OpenRouter API key required.  
  - *Version:* 1

- **Send a text message1 (Telegram)**  
  - *Type:* Telegram node  
  - *Role:* Sends the simplified agenda message to Telegram.  
  - *Configuration:* Similar to Send a text message node.  
  - *Credentials:* Telegram Bot token and chat_id required.  
  - *Edge Cases:* Same as Send a text message node.  
  - *Version:* 1.2

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                              | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                       |
|------------------------|----------------------------------|----------------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                 | Triggers workflow daily at 6 AM               | None                  | Set Timezone             | Runs workflows based on a schedule, with the default setting being 6 AM.                                          |
| Set Timezone           | Set                             | Sets default timezone (UTC) for event queries | Schedule Trigger      | Get many events          | Sets the timezone so that Google Calendar events are retrieved according to local time.                           |
| Get many events        | Google Calendar                 | Retrieves calendar events for today and tomorrow | Set Timezone          | If                       | Fetches all events from Google Calendar for today and tomorrow.                                                  |
| If                     | If                             | Checks if event has a location                 | Get many events        | HTTP Request (true), AI Agent2 (false) | Checks whether each event has a location.                                                                       |
| HTTP Request           | HTTP Request                   | Fetches weather data from WeatherAPI          | If (true)              | AI Agent                 | Fetch local weather data from WeatherAPI, based on event location, date, and time (customizable). Combine event and weather data into a single personalized message |
| AI Agent               | Langchain Agent (AI)            | Generates agenda summary with weather info    | HTTP Request           | Send a text message      |                                                                                                                  |
| OpenRouter Chat Model  | Langchain OpenRouter Chat Model | Provides Google Gemini 2.0 model for AI Agent | AI Agent                | AI Agent                 | Uses Google Gemini 2.0 Flash via OpenRouter                                                                     |
| Send a text message    | Telegram                       | Sends weather + agenda summary via Telegram   | AI Agent                | None                     |                                                                                                                  |
| AI Agent2              | Langchain Agent (AI)            | Generates agenda summary without weather info | If (false)              | Send a text message1     | If the event does not have a location, the data will still be presented in the form of a personalized message.    |
| OpenRouter Chat Model2 | Langchain OpenRouter Chat Model | Provides Google Gemini 2.0 model for AI Agent2 | AI Agent2               | AI Agent2                |                                                                                                                  |
| Send a text message1   | Telegram                       | Sends agenda summary without weather via Telegram | AI Agent2               | None                     |                                                                                                                  |
| Sticky Note            | Sticky Note                    | Workflow description and setup instructions   | None                   | None                     | See detailed content in section 5                                                                                 |
| Sticky Note1           | Sticky Note                    | Summarizes schedule, timezone, calendar fetch | None                   | None                     | Runs workflows based on a schedule, with the default setting being 6 AM. Sets the timezone so that Google Calendar events are retrieved according to local time. Fetches all events from Google Calendar for today and tomorrow. Checks whether each event has a location. |
| Sticky Note2           | Sticky Note                    | Summarizes weather data retrieval and combination | None                   | None                     | Fetch local weather data from WeatherAPI, based on event location, date, and time (customizable). Combine event and weather data into a single personalized message |
| Sticky Note3           | Sticky Note                    | Notes agenda generation if no location present | None                   | None                     | If the event does not have a location, the data will still be presented in the form of a personalized message.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 6:00 AM (set `triggerAtHour` to 6).  
   - No credentials needed.  

2. **Add Set Timezone Node**  
   - Type: Set  
   - Add a string field `Timezone` with value `"UTC"` (change as needed).  
   - Connect Schedule Trigger output to this node’s input.  

3. **Add Google Calendar Node ("Get many events")**  
   - Type: Google Calendar  
   - Operation: Get all events  
   - Set `timeMin` to `={{ $now.setZone($json.Timezone) }}`  
   - Set `timeMax` to `={{ $now.setZone($json.Timezone).plus({ day: 1 }) }}`  
   - Calendar: Select “primary” or specify calendar ID.  
   - Connect Set Timezone output to this node’s input.  
   - Add OAuth2 credentials for Google Calendar.  

4. **Add If Node**  
   - Type: If  
   - Condition: Check if `location` field is not empty:  
     - Left Value: `={{ $json.location }}`  
     - Operator: `notEmpty`  
   - Connect Get many events output to If node input.  

5. **Add HTTP Request Node (for weather)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `http://api.weatherapi.com/v1/current.json`  
   - Query Parameters:  
     - `key`: Your WeatherAPI key (replace placeholder)  
     - `q`: `={{ $json.location }}`  
     - `dt`: `={{ $json['time start'].split('T')[0] }}`  
     - `hour`: `={{ parseInt($json['time start'].split('T')[1].split(':')[0]) }}`  
   - Connect If node’s **true** output to this node.  

6. **Add AI Agent Node (with weather)**  
   - Type: Langchain Agent  
   - Text prompt: Use template that includes event agenda and weather parameters (temperature, humidity, UV index, wind, visibility).  
   - System message: Instruct polite, concise, emoji-enhanced plain text without punctuation.  
   - Connect HTTP Request output to this node.  

7. **Add OpenRouter Chat Model Node**  
   - Type: Langchain OpenRouter Chat Model  
   - Model: Set to `"google/gemini-2.0-flash-exp:free"`  
   - Connect as AI model input to AI Agent node.  
   - Add OpenRouter credentials in n8n.  

8. **Add Telegram Node ("Send a text message")**  
   - Type: Telegram  
   - Text: `={{ $json.output }}` (use AI Agent output)  
   - chatId: Your Telegram chat ID  
   - webhookId: Your Telegram webhook ID  
   - Disable append attribution.  
   - Connect AI Agent output to this node.  

9. **Add AI Agent2 Node (without weather)**  
   - Type: Langchain Agent  
   - Text prompt: Template with event summary and start/end date only (no weather).  
   - System message: Similar polite, simple instructions with emojis.  
   - Connect If node’s **false** output here.  

10. **Add OpenRouter Chat Model2 Node**  
    - Same as OpenRouter Chat Model node. Connect as AI model input to AI Agent2.  

11. **Add Telegram Node ("Send a text message1")**  
    - Similar to previous Telegram node.  
    - Text set to AI Agent2 output.  
    - Connect AI Agent2 output here.  

12. **Final Checks**  
    - Replace all placeholders (`YOUR_WEATHERAPI_KEY`, Telegram chatId, webhookId, credentials) with your actual values.  
    - Verify OAuth2 credentials for Google Calendar and OpenRouter API keys are configured in n8n.  
    - Activate workflow to enable daily automated messaging.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow automates personalized daily agenda delivery combining Google Calendar events and local weather conditions, summarized by AI and sent via Telegram.                                                                                                                                                                                            | Workflow description (Sticky Note)                      |
| Requires credentials setup: Google Calendar OAuth2, WeatherAPI key ([weatherapi.com](https://www.weatherapi.com/)), OpenRouter API key ([openrouter.ai](https://openrouter.ai/)), and Telegram Bot token and chat ID.                                                                                                                                             | Setup prerequisites                                     |
| Adjust timezone in Set Timezone node to match user local time zone for accurate event querying. Default is UTC.                                                                                                                                                                                                                                              | Configuration note                                      |
| The AI Agents use Google Gemini 2.0 Flash model via OpenRouter for natural language summarization, generating polite, concise, emoji-enhanced messages.                                                                                                                                                                                                      | AI model selection                                     |
| Telegram nodes require valid bot token and chat_id to send messages successfully. Ensure bot is authorized to send messages to that chat.                                                                                                                                                                                                                   | Telegram integration                                    |
| If no event location is available, the workflow still sends a simplified agenda message without weather data.                                                                                                                                                                                                                                               | Functional fallback                                     |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.