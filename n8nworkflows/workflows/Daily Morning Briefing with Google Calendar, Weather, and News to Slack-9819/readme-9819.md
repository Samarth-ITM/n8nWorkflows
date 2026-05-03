Daily Morning Briefing with Google Calendar, Weather, and News to Slack

https://n8nworkflows.xyz/workflows/daily-morning-briefing-with-google-calendar--weather--and-news-to-slack-9819


# Daily Morning Briefing with Google Calendar, Weather, and News to Slack

---

### 1. Workflow Overview

This workflow automates a **Personal Daily Morning Briefing** by aggregating key information and sending it as a single message to a Slack channel every morning at 7 AM. It targets busy professionals who want a concise daily update combining calendar events, weather forecast, and top news headlines.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Configuration Setup:** Initiates the workflow daily at 7 AM and sets key configuration parameters such as RSS feed URL and weather API endpoint.

- **1.2 Data Retrieval:** Fetches data concurrently from three sources:
  - Google Calendar for today’s events
  - Weather API for current weather in Tokyo
  - RSS feed for top news headlines

- **1.3 Data Aggregation & Formatting:** Waits for all data fetches to complete, then formats the data into a structured briefing message.

- **1.4 Message Delivery:** Sends the formatted briefing message to a specified Slack channel using OAuth2 authentication.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration Setup

- **Overview:**  
  This block triggers the workflow every day at 7 AM and defines URLs for the weather API and news RSS feed. It centralizes the configuration for easy adjustments.

- **Nodes Involved:**  
  - Daily Morning Trigger  
  - Workflow Configuration

- **Node Details:**

  - **Daily Morning Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts workflow execution daily at a fixed hour.  
    - *Configuration:* Trigger set at 7:00 AM local time (hour-only interval).  
    - *Inputs:* None  
    - *Outputs:* Connected to Workflow Configuration  
    - *Edge Cases:* Workflow will not run if n8n instance is offline at trigger time; time zone considerations may affect trigger time accuracy.

  - **Workflow Configuration**  
    - *Type:* Set  
    - *Role:* Stores static configuration variables used downstream.  
    - *Configuration:*  
      - `rssUrl` set to Google News Japan RSS (Japanese news).  
      - `weatherApiUrl` set to `https://wttr.in/Tokyo?format=j1` (JSON weather forecast for Tokyo).  
    - *Inputs:* From Daily Morning Trigger  
    - *Outputs:* Feeds three parallel data retrieval nodes  
    - *Edge Cases:* URLs hardcoded; changing feed or city requires manual update here.

#### 2.2 Data Retrieval

- **Overview:**  
  Concurrently obtains data from Google Calendar, weather API, and news RSS feed.

- **Nodes Involved:**  
  - Get Today's Calendar Events  
  - Get Weather Forecast  
  - Get Top News from RSS

- **Node Details:**

  - **Get Today's Calendar Events**  
    - *Type:* Google Calendar  
    - *Role:* Retrieves all calendar events scheduled for the current day.  
    - *Configuration:*  
      - Limits to 10 events max.  
      - Filters events between start and end of the current day using expressions (`$now.startOf('day')`, `$now.endOf('day')`).  
      - Uses OAuth2 credentials for Google Calendar (requires prior OAuth setup).  
      - Calendar account selected from credential list.  
    - *Inputs:* From Workflow Configuration  
    - *Outputs:* Connected to Wait for All Data node  
    - *Edge Cases:*  
      - OAuth token expiration or auth failure.  
      - No events found returns empty array.  
      - API rate limits could delay or block requests.

  - **Get Weather Forecast**  
    - *Type:* HTTP Request  
    - *Role:* Fetches current weather data for Tokyo from wttr.in.  
    - *Configuration:*  
      - URL dynamically read from Workflow Configuration node (`weatherApiUrl`).  
      - Expects JSON response (`responseFormat: json`).  
    - *Inputs:* From Workflow Configuration  
    - *Outputs:* Connected to Wait for All Data node  
    - *Edge Cases:*  
      - Network issues or API downtime.  
      - Unexpected JSON format changes.  
      - No authentication required.

  - **Get Top News from RSS**  
    - *Type:* RSS Feed Read  
    - *Role:* Reads latest news headlines from a Google News RSS feed.  
    - *Configuration:*  
      - URL dynamically set from Workflow Configuration node (`rssUrl`).  
      - Default RSS parsing options.  
    - *Inputs:* From Workflow Configuration  
    - *Outputs:* Connected to Wait for All Data node  
    - *Edge Cases:*  
      - RSS feed changes or becomes unavailable.  
      - Parsing errors due to malformed feed.

#### 2.3 Data Aggregation & Formatting

- **Overview:**  
  Waits for all three data sources to complete, then constructs a structured briefing message combining calendar events, weather, and news.

- **Nodes Involved:**  
  - Wait for All Data  
  - Format Briefing Message

- **Node Details:**

  - **Wait for All Data**  
    - *Type:* Merge  
    - *Role:* Synchronizes data by waiting for all three input streams (calendar, weather, news) to finish before continuing.  
    - *Configuration:*  
      - Number of inputs set to 3.  
      - Merge mode defaults to wait for all.  
    - *Inputs:* Receives data from the three data retrieval nodes.  
    - *Outputs:* Single combined flow to Format Briefing Message.  
    - *Edge Cases:*  
      - If any input fails or is delayed, downstream steps wait indefinitely or until timeout.

  - **Format Briefing Message**  
    - *Type:* Set  
    - *Role:* Builds the Slack message content using expressions to format each data block.  
    - *Configuration:*  
      - Creates a single string variable `briefingMessage` formatted in Slack markdown, including:  
        - Calendar events with time formatted in Japanese locale or fallback message if none.  
        - Current weather description and temperature from weather API JSON.  
        - Top 3 news headlines as clickable links or fallback message if none.  
      - Uses JavaScript expressions to map and format arrays from previous nodes.  
    - *Inputs:* From Wait for All Data  
    - *Outputs:* Connected to Post to Slack node  
    - *Edge Cases:*  
      - Expression errors if expected JSON structure changes.  
      - Empty data arrays produce fallback messages.

#### 2.4 Message Delivery

- **Overview:**  
  Sends the final formatted briefing message to a designated Slack channel using OAuth2 authentication.

- **Nodes Involved:**  
  - Post to Slack

- **Node Details:**

  - **Post to Slack**  
    - *Type:* Slack  
    - *Role:* Posts the briefing message text to a specified Slack channel.  
    - *Configuration:*  
      - Message text set via expression from `Format Briefing Message` node’s `briefingMessage`.  
      - Channel selected from a predefined list (`C09M5L3UB8U` as channel ID).  
      - Authentication via OAuth2 credentials for Slack.  
    - *Inputs:* From Format Briefing Message  
    - *Outputs:* None (end of workflow)  
    - *Edge Cases:*  
      - OAuth token expiration or permission errors.  
      - Slack API rate limits or connectivity issues.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                               | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                  |
|---------------------------|--------------------|-----------------------------------------------|--------------------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| Daily Morning Trigger      | Schedule Trigger   | Starts workflow daily at 7 AM                  | None                           | Workflow Configuration         |                                                                                             |
| Workflow Configuration     | Set                | Defines RSS and weather API URLs               | Daily Morning Trigger          | Get Today's Calendar Events, Get Weather Forecast, Get Top News from RSS |                                                                                             |
| Get Today's Calendar Events| Google Calendar    | Fetches today’s calendar events                 | Workflow Configuration         | Wait for All Data              |                                                                                             |
| Get Weather Forecast       | HTTP Request       | Gets current weather for Tokyo                   | Workflow Configuration         | Wait for All Data              | Fetches current weather data from wttr.in (Tokyo).                                         |
| Get Top News from RSS      | RSS Feed Read      | Reads top news headlines                         | Workflow Configuration         | Wait for All Data              | Reads top 3 headlines from Google News RSS feed.                                           |
| Wait for All Data          | Merge              | Waits for all data retrievals to complete       | Get Today's Calendar Events, Get Weather Forecast, Get Top News from RSS | Format Briefing Message       |                                                                                             |
| Format Briefing Message    | Set                | Formats combined data into Slack message         | Wait for All Data              | Post to Slack                 |                                                                                             |
| Post to Slack              | Slack              | Sends briefing message to Slack channel         | Format Briefing Message        | None                          | Sends the compiled morning briefing message to your selected Slack channel.                 |
| Template Description       | Sticky Note        | Documentation for the workflow                   | None                         | None                          | ## Personal Daily Morning Briefing Automation... (full description and setup instructions)  |
| Weather Note              | Sticky Note        | Notes on weather data source                      | None                         | None                          | Fetches current weather data from wttr.in (Tokyo).                                         |
| News Note                  | Sticky Note        | Notes on news data source                         | None                         | None                          | Reads top 3 headlines from Google News RSS feed.                                           |
| Slack Note                 | Sticky Note        | Notes on Slack message sending                    | None                         | None                          | Sends the compiled morning briefing message to your selected Slack channel.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set trigger time to daily at 7:00 AM (hour interval at 7).  
   - This node has no input.

2. **Create Set Node for Workflow Configuration:**  
   - Type: Set  
   - Add two string fields:  
     - `rssUrl` with value: `https://news.google.com/rss?hl=ja&gl=JP&ceid=JP:ja`  
     - `weatherApiUrl` with value: `https://wttr.in/Tokyo?format=j1`  
   - Connect Schedule Trigger output to this node.

3. **Create Google Calendar Node for Events:**  
   - Type: Google Calendar  
   - Operation: Get All Events  
   - Limit: 10  
   - Time Min: `={{ $now.startOf('day') }}`  
   - Time Max: `={{ $now.endOf('day') }}`  
   - Select your Google Calendar via OAuth2 credential.  
   - Connect Workflow Configuration output to this node.

4. **Create HTTP Request Node for Weather:**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $('Workflow Configuration').first().json.weatherApiUrl }}`  
   - Response Format: JSON  
   - Connect Workflow Configuration output to this node.

5. **Create RSS Feed Read Node for News:**  
   - Type: RSS Feed Read  
   - URL: `={{ $('Workflow Configuration').first().json.rssUrl }}`  
   - Connect Workflow Configuration output to this node.

6. **Create Merge Node to Wait for All Data:**  
   - Type: Merge  
   - Number of Inputs: 3  
   - Default mode (wait for all inputs)  
   - Connect outputs of Google Calendar, HTTP Request, and RSS Feed Read nodes to this node’s inputs.

7. **Create Set Node to Format Message:**  
   - Type: Set  
   - Add string field `briefingMessage` with the following Slack markdown template (using expressions):  
     ```
     =*📅 Daily Morning Briefing*

     *📆 今日の予定*
     {{ $('Get Today\'s Calendar Events').all().length > 0 ? $('Get Today\'s Calendar Events').all().map(event => '• ' + event.json.summary + ' - ' + new Date(event.json.start.dateTime).toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' })).join('\n') : '今日の予定はありません' }}

     *🌤️ 天気予報*
     • {{ $('Get Weather Forecast').first().json.current_condition[0].weatherDesc[0].value }} / 気温: {{ $('Get Weather Forecast').first().json.current_condition[0].temp_C }}°C

     *📰 トップニュース*
     {{ $('Get Top News from RSS').all().length > 0 ? $('Get Top News from RSS').all().slice(0, 3).map(article => '• <' + article.json.link + '|' + article.json.title + '>').join('\n') : 'ニュースはありません' }}
     ```
   - Connect Merge node output to this node.

8. **Create Slack Node to Post Message:**  
   - Type: Slack  
   - Authentication: OAuth2 (configure Slack credentials beforehand)  
   - Channel: Select your target Slack channel (by channel ID or list select)  
   - Text: `={{ $('Format Briefing Message').first().json.briefingMessage }}`  
   - Connect Format Briefing Message node output to this node.

9. **Test Workflow:**  
   - Activate workflow and manually trigger or wait for the 7 AM schedule to confirm the message posts correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Personal Daily Morning Briefing Automation** is designed for busy professionals needing quick updates combining calendar, weather, and news. | See sticky note "Template Description" in the workflow for full details.                                                                                        |
| Weather data is fetched from [wttr.in](https://wttr.in) for Tokyo without requiring authentication.                                  | Weather Note sticky node.                                                                                                                                         |
| News headlines come from Google News Japan RSS feed at `https://news.google.com/rss?hl=ja&gl=JP&ceid=JP:ja`.                          | News Note sticky node.                                                                                                                                           |
| Slack messages use OAuth2 authentication; ensure your Slack app has proper scopes for posting messages to channels.                   | Slack Note sticky node.                                                                                                                                           |
| To customize, modify trigger time, RSS URL, weather city in the Workflow Configuration node, or adjust message formatting in the Set node. | Template Description sticky note.                                                                                                                               |

---

*Disclaimer: The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.*