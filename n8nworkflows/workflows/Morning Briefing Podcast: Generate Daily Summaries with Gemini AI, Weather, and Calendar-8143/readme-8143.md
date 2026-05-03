Morning Briefing Podcast: Generate Daily Summaries with Gemini AI, Weather, and Calendar

https://n8nworkflows.xyz/workflows/morning-briefing-podcast--generate-daily-summaries-with-gemini-ai--weather--and-calendar-8143


# Morning Briefing Podcast: Generate Daily Summaries with Gemini AI, Weather, and Calendar

### 1. Workflow Overview

This workflow automates the generation of a daily "Morning Briefing Podcast" by aggregating personalized weather forecasts, calendar events, and technology news headlines, then synthesizing these inputs into a natural conversational script using Google Gemini AI. It culminates in generating an audio podcast file and sending it to a Telegram channel. The workflow is designed for daily execution and targets users who want a concise, engaging morning briefing combining relevant personal and global information.

Logical blocks:

- **1.1 Initialization and Configuration**: User-triggered workflow start and setting user-specific parameters.
- **1.2 Weather Data Fetching and Summary Generation**: Retrieves 15-hour weather forecast and generates a conversational weather segment.
- **1.3 Calendar Events Retrieval and Summary Generation**: Fetches user's daily calendar events and generates a conversational schedule segment.
- **1.4 News Headlines Retrieval and Summary Generation**: Collects tech news headlines from multiple sources, processes them, and generates a conversational news segment.
- **1.5 Podcast Assembly and Audio Generation**: Aggregates AI-generated script parts, synthesizes audio using Gemini TTS, converts and saves audio files, and sends the final podcast audio to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration

**Overview:**  
Starts the workflow manually and sets user-specific settings such as user name, location coordinates, city name, and preferred output language.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Config  
- Route

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow.  
  - *Config:* No parameters; triggers workflow execution.  
  - *Input:* None  
  - *Output:* Connects to Config node.  
  - *Edge cases:* None significant; manual start only.

- **Config**  
  - *Type:* Set  
  - *Role:* Defines static user configuration parameters used throughout the workflow.  
  - *Parameters:*  
    - `user_name`: "Atta"  
    - `city_name`: "Breda"  
    - `city_lat`: "51.5719"  
    - `city_lon`: "4.7683"  
    - `output_language`: "English (US)"  
  - *Input:* From Manual Trigger  
  - *Output:* To Route node  
  - *Edge cases:* None; static data, but user can modify for personalization.

- **Route**  
  - *Type:* NoOp (No Operation)  
  - *Role:* Acts as a logical splitter to branch the workflow into parallel data retrievals: calendar, weather, and news.  
  - *Input:* From Config node  
  - *Output:* To Get Today Meetings, Open Weather Map, and News Sources nodes  
  - *Edge cases:* None

---

#### 2.2 Weather Data Fetching and Summary Generation

**Overview:**  
Retrieves a 15-hour weather forecast from OpenWeatherMap API using configured location and generates a conversational weather segment script via AI.

**Nodes Involved:**  
- Open Weather Map  
- Gemini_Weather  
- Weather Summary  
- Sticky Note (User location weather summary)

**Node Details:**

- **Open Weather Map**  
  - *Type:* HTTP Request  
  - *Role:* Fetches 5 forecast data points (~15 hours) for the configured city latitude and longitude from OpenWeatherMap API.  
  - *Parameters:*  
    - URL: `https://api.openweathermap.org/data/2.5/forecast`  
    - Query: lat, lon, units (metric), cnt (5)  
    - Authentication: HTTP Query Auth with API key  
  - *Input:* From Route node  
  - *Output:* To Weather Summary node  
  - *Edge cases:* API key invalid, rate limits, network failures, location data invalid.

- **Gemini_Weather**  
  - *Type:* Google Gemini Language Model Chat  
  - *Role:* Invoked internally by Weather Summary agent node (AI language model call); not directly connected in the workflow JSON but required for AI processing.  
  - *Input:* N/A (used internally)  
  - *Output:* N/A  
  - *Credentials:* Google Palm API  
  - *Edge cases:* API quota, model availability.

- **Weather Summary**  
  - *Type:* LangChain Agent (AI Language Model)  
  - *Role:* Uses the weather forecast JSON data to generate a friendly, brief conversational podcast opening segment, including a personalized greeting, weather details, and high temperature/time.  
  - *Parameters:*  
    - Text input includes user name, current date, and full weather list data serialized as JSON.  
    - System message instructs AI to produce a two-host dialog with specific style and language.  
  - *Input:* From Open Weather Map  
  - *Output:* To Merge node (input 0)  
  - *Edge cases:* AI response errors, empty or malformed weather data, expression failures in input text.

- **Sticky Note (User location weather summary)**  
  - *Content:* Explains this workflow part fetches weather and generates conversational summary with AI.  
  - *Context:* Documentation aid.

---

#### 2.3 Calendar Events Retrieval and Summary Generation

**Overview:**  
Retrieves all calendar events for the current day from the user's Google Calendar and generates a conversational summary segment for the podcast.

**Nodes Involved:**  
- Get Today Meetings  
- Aggregate Events  
- Calendar Summary  
- Gemini_Calendar  
- Sticky Note1 (User Calendar summary)

**Node Details:**

- **Get Today Meetings**  
  - *Type:* Google Calendar  
  - *Role:* Fetches all calendar events from start to end of the current day for the configured calendar email.  
  - *Parameters:*  
    - timeMin: start of current day  
    - timeMax: end of current day  
    - Return all events (not paginated)  
  - *Credentials:* Google Calendar OAuth2  
  - *Input:* From Route node  
  - *Output:* To Aggregate Events  
  - *Edge cases:* OAuth token expiration, no events found, API limits.

- **Aggregate Events**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all calendar event items into a single collection under `events`.  
  - *Input:* From Get Today Meetings  
  - *Output:* To Calendar Summary  
  - *Edge cases:* Empty event list.

- **Calendar Summary**  
  - *Type:* LangChain Agent (AI Language Model)  
  - *Role:* Generates a podcast script segment summarizing the user's calendar events for the day based on the aggregated events data.  
  - *Parameters:*  
    - Text input includes serialized calendar events JSON.  
    - System message instructs AI to produce a middle podcast segment dialogue with handling for empty or busy schedules per instructions.  
  - *Input:* From Aggregate Events  
  - *Output:* To Merge node (input 1)  
  - *Edge cases:* AI errors, empty or malformed data, expression failures.

- **Gemini_Calendar**  
  - *Type:* Google Gemini Language Model Chat  
  - *Role:* Used internally by Calendar Summary node for AI processing.  
  - *Credentials:* Google Palm API  
  - *Edge cases:* API quota, response errors.

- **Sticky Note1 (User Calendar summary)**  
  - *Content:* Describes this block fetches user meetings/events and generates a conversational summary.  
  - *Context:* Documentation aid.

---

#### 2.4 News Headlines Retrieval and Summary Generation

**Overview:**  
Fetches top technology news headlines from multiple sources via NewsAPI, extracts useful fields, aggregates them, and generates a conversational news segment.

**Nodes Involved:**  
- News Sources  
- Split Out News Sources  
- Get Headlines from NewsApi  
- Split Out  
- Get Useful Fields  
- Aggregate Headlines  
- News Summary  
- Gemini_News  
- Sticky Note2 (News summary)

**Node Details:**

- **News Sources**  
  - *Type:* Set  
  - *Role:* Defines an array of technology news source identifiers to query NewsAPI.  
  - *Parameters:* List of 10 sources including "techcrunch", "wired", "the-verge", etc.  
  - *Input:* From Route node  
  - *Output:* To Split Out News Sources  
  - *Edge cases:* None; user-editable sources.

- **Split Out News Sources**  
  - *Type:* Split Out  
  - *Role:* Splits the array of news sources into individual items for sequential processing.  
  - *Input:* From News Sources  
  - *Output:* To Get Headlines from NewsApi  
  - *Edge cases:* Empty source list.

- **Get Headlines from NewsApi**  
  - *Type:* HTTP Request  
  - *Role:* Fetches top headlines (3 per source) from NewsAPI for the previous day, filtering by source.  
  - *Parameters:*  
    - URL: `https://newsapi.org/v2/top-headlines`  
    - Query includes `from` date (yesterday), `pageSize`=3, and `sources` as current source item.  
    - Authentication: HTTP Header Auth with NewsAPI key.  
  - *Input:* From Split Out News Sources  
  - *Output:* To Split Out  
  - *Edge cases:* API key invalid, rate limits, network errors.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the articles array from NewsAPI response into individual article items.  
  - *Input:* From Get Headlines from NewsApi  
  - *Output:* To Get Useful Fields  
  - *Edge cases:* Empty articles list.

- **Get Useful Fields**  
  - *Type:* Set  
  - *Role:* Extracts key fields (`source.name`, `title`, `description`) from each article for simplified data.  
  - *Input:* From Split Out  
  - *Output:* To Aggregate Headlines  
  - *Edge cases:* Missing fields in articles.

- **Aggregate Headlines**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all processed article data into a single `headlines` collection.  
  - *Input:* From Get Useful Fields  
  - *Output:* To News Summary  
  - *Edge cases:* Empty aggregate.

- **News Summary**  
  - *Type:* LangChain Agent (AI Language Model)  
  - *Role:* Generates the final podcast news segment as a conversational exchange based on aggregated headlines.  
  - *Parameters:*  
    - Text input includes serialized `headlines` JSON.  
    - System message instructs AI to produce an engaging, concise news segment with introduction and podcast closing.  
  - *Input:* From Aggregate Headlines  
  - *Output:* To Merge node (input 2)  
  - *Edge cases:* AI errors, empty or malformed headlines.

- **Gemini_News**  
  - *Type:* Google Gemini Language Model Chat  
  - *Role:* Invoked internally by News Summary node for AI processing.  
  - *Credentials:* Google Palm API  
  - *Edge cases:* API quota, response errors.

- **Sticky Note2 (News summary)**  
  - *Content:* Explains this block fetches top tech news headlines and generates a conversational summary.  
  - *Context:* Documentation aid.

---

#### 2.5 Podcast Assembly and Audio Generation

**Overview:**  
Merges the three podcast script segments (weather, calendar, news), generates speech audio using Google Gemini TTS API with multi-speaker voices, converts and saves the audio file, then sends it to a Telegram channel.

**Nodes Involved:**  
- Merge  
- Aggregate Podcast Parts  
- Set Filename  
- Generate Podcast Audio  
- Convert Audio to File  
- Write Audio File on Disk  
- Convert Audio to MP3  
- Read Audio File from Disk  
- Send Podcast to Telegram

**Node Details:**

- **Merge**  
  - *Type:* Merge  
  - *Role:* Merges three inputs (weather summary, calendar summary, news summary) into one stream for final aggregation.  
  - *Parameters:* `numberInputs=3`  
  - *Input:* Weather Summary (0), Calendar Summary (1), News Summary (2)  
  - *Output:* To Aggregate Podcast Parts  
  - *Edge cases:* Missing input segments; merge may wait indefinitely.

- **Aggregate Podcast Parts**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all merged podcast segments into a single `output` collection for TTS processing.  
  - *Input:* From Merge  
  - *Output:* To Set Filename  
  - *Edge cases:* Empty aggregation.

- **Set Filename**  
  - *Type:* Set  
  - *Role:* Creates a unique output filename using current ISO timestamp (e.g., "goodmorning-2024-06-07T...").  
  - *Input:* From Aggregate Podcast Parts  
  - *Output:* To Generate Podcast Audio  
  - *Edge cases:* None.

- **Generate Podcast Audio**  
  - *Type:* HTTP Request  
  - *Role:* Sends the aggregated podcast script to Google Gemini TTS API to generate multi-speaker audio content.  
  - *Parameters:*  
    - URL: Gemini TTS endpoint with model `gemini-2.5-flash-preview-tts`  
    - POST body includes text parts with speaker tags matching the AI Assistant and Byte voices  
    - Response expected as audio data in base64 inline data  
    - Authentication: HTTP Header Auth with Gemini API key and HTTP Basic Auth for admin  
  - *Input:* From Set Filename  
  - *Output:* To Convert Audio to File  
  - *Edge cases:* API errors, network timeouts, authentication failures.

- **Convert Audio to File**  
  - *Type:* Convert To File  
  - *Role:* Converts TTS base64 inline audio data (PCM format) into binary file data for saving.  
  - *Parameters:*  
    - Filename from Set Filename node with `.pcm` extension  
    - MIME type from TTS response metadata  
  - *Input:* From Generate Podcast Audio  
  - *Output:* To Write Audio File on Disk  
  - *Edge cases:* Data format issues.

- **Write Audio File on Disk**  
  - *Type:* Read/Write File  
  - *Role:* Writes the binary PCM audio file to the local disk under user directory `/users/attaks/`.  
  - *Parameters:*  
    - Filename from Set Filename with `.pcm` extension  
    - Operation: write  
  - *Input:* From Convert Audio to File  
  - *Output:* To Convert Audio to MP3  
  - *Edge cases:* Disk write permission errors.

- **Convert Audio to MP3**  
  - *Type:* Execute Command  
  - *Role:* Uses `ffmpeg` CLI to convert the PCM audio file to MP3 format with 192 kbps bitrate.  
  - *Parameters:*  
    - Command template includes source and destination filename using Set Filename variables  
  - *Input:* From Write Audio File on Disk  
  - *Output:* To Read Audio File from Disk  
  - *Edge cases:* FFMPEG not installed, command failures.

- **Read Audio File from Disk**  
  - *Type:* Read/Write File  
  - *Role:* Reads the generated MP3 audio file from disk to prepare for sending.  
  - *Parameters:*  
    - File selector uses the MP3 filename from Set Filename  
  - *Input:* From Convert Audio to MP3  
  - *Output:* To Send Podcast to Telegram  
  - *Edge cases:* File not found.

- **Send Podcast to Telegram**  
  - *Type:* Telegram  
  - *Role:* Sends the MP3 podcast audio as a message to a configured Telegram chat/channel.  
  - *Parameters:*  
    - Chat ID: `-4917576370` (a Telegram group or channel)  
    - Operation: `sendAudio` with binary data enabled  
    - Title includes "Good Morning" and current date  
  - *Credentials:* Telegram API for GoodMorning Bot  
  - *Input:* From Read Audio File from Disk  
  - *Output:* None (end of workflow)  
  - *Edge cases:* Invalid chat ID, bot not authorized, network errors.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                  | Input Node(s)                           | Output Node(s)                        | Sticky Note                                                                                          |
|-------------------------|----------------------------------|-------------------------------------------------|---------------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Entry point, manual start                        | None                                  | Config                              |                                                                                                    |
| Config                  | Set                              | Defines user location, name, language           | When clicking ‘Execute workflow’      | Route                              |                                                                                                    |
| Route                   | NoOp                             | Branches to weather, calendar, news retrieval   | Config                               | Get Today Meetings, Open Weather Map, News Sources |                                                                                                    |
| Open Weather Map         | HTTP Request                    | Fetches 15-hour weather forecast                  | Route                               | Weather Summary                    |                                                                                                    |
| Gemini_Weather          | Google Gemini LM Chat            | AI model for weather summary generation           | N/A (used internally)                 | N/A                                |                                                                                                    |
| Weather Summary          | LangChain Agent                 | Generates conversational weather segment          | Open Weather Map                    | Merge (input 0)                    | User location weather summary: retrieves weather and generates conversational summary              |
| Get Today Meetings       | Google Calendar                 | Retrieves today's calendar events                  | Route                               | Aggregate Events                  |                                                                                                    |
| Aggregate Events         | Aggregate                       | Aggregates calendar events into a collection      | Get Today Meetings                  | Calendar Summary                  |                                                                                                    |
| Gemini_Calendar          | Google Gemini LM Chat            | AI model for calendar summary generation           | N/A (used internally)                 | N/A                                |                                                                                                    |
| Calendar Summary         | LangChain Agent                 | Generates conversational calendar segment          | Aggregate Events                   | Merge (input 1)                   | User Calendar summary: gathers meetings/events and generates conversational summary                 |
| News Sources             | Set                              | Defines list of news sources                        | Route                               | Split Out News Sources            |                                                                                                    |
| Split Out News Sources   | Split Out                      | Splits news sources array into individual sources | News Sources                      | Get Headlines from NewsApi        |                                                                                                    |
| Get Headlines from NewsApi | HTTP Request                    | Fetches news headlines from NewsAPI                | Split Out News Sources             | Split Out                        |                                                                                                    |
| Split Out               | Split Out                      | Splits news articles array into individual articles | Get Headlines from NewsApi          | Get Useful Fields                |                                                                                                    |
| Get Useful Fields        | Set                              | Extracts source, title, description from articles  | Split Out                        | Aggregate Headlines              |                                                                                                    |
| Aggregate Headlines      | Aggregate                       | Aggregates all news headlines into one collection  | Get Useful Fields                 | News Summary                    |                                                                                                    |
| Gemini_News              | Google Gemini LM Chat            | AI model for news summary generation               | N/A (used internally)                 | N/A                                |                                                                                                    |
| News Summary             | LangChain Agent                 | Generates conversational news segment               | Aggregate Headlines              | Merge (input 2)                  | News summary: collects headlines and generates conversational summary                             |
| Merge                   | Merge                           | Merges weather, calendar, news AI outputs          | Weather Summary, Calendar Summary, News Summary | Aggregate Podcast Parts           |                                                                                                    |
| Aggregate Podcast Parts  | Aggregate                       | Aggregates merged script segments                   | Merge                            | Set Filename                    |                                                                                                    |
| Set Filename             | Set                              | Generates unique filename for audio output          | Aggregate Podcast Parts           | Generate Podcast Audio          |                                                                                                    |
| Generate Podcast Audio   | HTTP Request                    | Sends text to Gemini TTS API, generates audio       | Set Filename                    | Convert Audio to File           |                                                                                                    |
| Convert Audio to File    | Convert To File                 | Converts base64 audio data to binary file           | Generate Podcast Audio           | Write Audio File on Disk        |                                                                                                    |
| Write Audio File on Disk | Read/Write File                | Saves PCM audio file to disk                         | Convert Audio to File            | Convert Audio to MP3            |                                                                                                    |
| Convert Audio to MP3     | Execute Command                | Converts PCM file to MP3 using ffmpeg               | Write Audio File on Disk         | Read Audio File from Disk       |                                                                                                    |
| Read Audio File from Disk | Read/Write File                | Reads MP3 audio file from disk                       | Convert Audio to MP3             | Send Podcast to Telegram        |                                                                                                    |
| Send Podcast to Telegram | Telegram                       | Sends final MP3 podcast audio to Telegram channel   | Read Audio File from Disk        | None                           |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: To manually start the workflow.  
   - No special configuration.

2. **Create a Set Node for User Configuration**  
   - Name: "Config"  
   - Add fields:  
     - `user_name` (string): e.g., "Atta"  
     - `city_name` (string): e.g., "Breda"  
     - `city_lat` (string): e.g., "51.5719"  
     - `city_lon` (string): e.g., "4.7683"  
     - `output_language` (string): e.g., "English (US)"  
   - Connect from Manual Trigger.

3. **Create a NoOp Node as Route**  
   - Name: "Route"  
   - Connect from Config.  
   - Purpose: Logical branching point.

4. **Weather Data Retrieval and Summary**  
   - Create HTTP Request Node: "Open Weather Map"  
     - URL: `https://api.openweathermap.org/data/2.5/forecast`  
     - Query params: lat (`{{$json.city_lat}}`), lon (`{{$json.city_lon}}`), units = "metric", cnt = 5  
     - Authentication: HTTP Query Auth with OpenWeatherMap API key.  
     - Connect from Route.  
   - Create LangChain Agent Node: "Weather Summary"  
     - Text input: Include `user_name`, current date, and weather forecast data serialized as JSON.  
     - System prompt: Instructions for a friendly weather podcast opening segment, with speaker tags and output language from Config.  
     - Connect from Open Weather Map node.  
     - Credential: Google Palm API for Gemini_Weather.  
   - Connect Weather Summary output to Merge node input 0.

5. **Calendar Events Retrieval and Summary**  
   - Create Google Calendar Node: "Get Today Meetings"  
     - Operation: Get all events  
     - Calendar: User’s Google Calendar email  
     - timeMin: start of current day (`{{$now.startOf('day')}}`)  
     - timeMax: end of current day (`{{$now.endOf('day')}}`)  
     - Credential: Google Calendar OAuth2.  
     - Connect from Route.  
   - Create Aggregate Node: "Aggregate Events"  
     - Aggregate all items into field `events`.  
     - Connect from Get Today Meetings.  
   - Create LangChain Agent Node: "Calendar Summary"  
     - Text input: Serialized `events` JSON from Aggregate Events.  
     - System prompt: Instructions to summarize calendar events as a middle podcast segment with speaker tags and output language.  
     - Credential: Google Palm API for Gemini_Calendar.  
     - Connect from Aggregate Events.  
   - Connect Calendar Summary output to Merge node input 1.

6. **News Headlines Retrieval and Summary**  
   - Create Set Node: "News Sources"  
     - Define array `news_sources` with top tech news site identifiers (e.g., techcrunch, wired, etc.).  
     - Connect from Route.  
   - Create Split Out Node: "Split Out News Sources"  
     - Field to split out: `news_sources`.  
     - Connect from News Sources.  
   - Create HTTP Request Node: "Get Headlines from NewsApi"  
     - URL: `https://newsapi.org/v2/top-headlines`  
     - Query params: `from` = yesterday's date, `pageSize`=3, `sources` = current source item (`{{$json.news_sources}}`).  
     - Authentication: HTTP Header Auth with NewsAPI key.  
     - Connect from Split Out News Sources.  
   - Create Split Out Node: "Split Out"  
     - Field to split out: `articles`.  
     - Connect from Get Headlines from NewsApi.  
   - Create Set Node: "Get Useful Fields"  
     - Extract `source.name`, `title`, `description` from each article.  
     - Connect from Split Out.  
   - Create Aggregate Node: "Aggregate Headlines"  
     - Aggregate all processed articles into `headlines`.  
     - Connect from Get Useful Fields.  
   - Create LangChain Agent Node: "News Summary"  
     - Text input: Serialized `headlines` JSON.  
     - System prompt: Instructions for final podcast news segment with speaker tags and output language.  
     - Credential: Google Palm API for Gemini_News.  
     - Connect from Aggregate Headlines.  
   - Connect News Summary output to Merge node input 2.

7. **Merge AI Outputs**  
   - Create Merge Node: "Merge"  
     - Set number of inputs to 3.  
     - Connect Weather Summary (input 0), Calendar Summary (input 1), News Summary (input 2) to Merge.

8. **Aggregate Podcast Parts**  
   - Create Aggregate Node: "Aggregate Podcast Parts"  
     - Aggregate all merged items into field `output`.  
     - Connect from Merge.

9. **Set Filename for Output**  
   - Create Set Node: "Set Filename"  
     - Define field `output_filename` as `goodmorning-{{$now.toISO()}}`.  
     - Connect from Aggregate Podcast Parts.

10. **Generate Podcast Audio**  
    - Create HTTP Request Node: "Generate Podcast Audio"  
      - URL: Gemini TTS endpoint `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent`  
      - Method: POST  
      - JSON Body: Includes aggregated podcast text with speaker tags for multi-speaker voice config (Fenrir for AI Assistant, Kore for Byte).  
      - Authentication: HTTP Header Auth and HTTP Basic Auth with Gemini API keys.  
      - Connect from Set Filename.

11. **Convert Audio to File**  
    - Create Convert To File Node: "Convert Audio to File"  
      - Source property: path to base64 audio data in TTS response  
      - File name: `{{$json.output_filename}}.pcm`  
      - MIME type from TTS response metadata  
      - Connect from Generate Podcast Audio.

12. **Write Audio File on Disk**  
    - Create Read/Write File Node: "Write Audio File on Disk"  
      - File Name: `/users/attaks/{{$json.output_filename}}.pcm`  
      - Operation: Write  
      - Connect from Convert Audio to File.

13. **Convert Audio to MP3**  
    - Create Execute Command Node: "Convert Audio to MP3"  
      - Command: `ffmpeg -y -f s16le -ar 24000 -ac 1 -i /users/attaks/{{$json.output_filename}}.pcm -b:a 192k /users/attaks/{{$json.output_filename}}.mp3`  
      - Connect from Write Audio File on Disk.  
      - Ensure ffmpeg is installed on the host system.

14. **Read MP3 Audio File from Disk**  
    - Create Read/Write File Node: "Read Audio File from Disk"  
      - File Selector: `/users/attaks/{{$json.output_filename}}.mp3`  
      - Operation: Read  
      - Connect from Convert Audio to MP3.

15. **Send Podcast Audio to Telegram**  
    - Create Telegram Node: "Send Podcast to Telegram"  
      - Operation: sendAudio  
      - Chat ID: `-4917576370` (replace with your Telegram chat/channel ID)  
      - Use binary data from previous node  
      - Credential: Telegram API Bot credentials  
      - Connect from Read Audio File from Disk.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini (PaLM) API for AI language and text-to-speech tasks requiring API keys.    | Requires Google Cloud account with PaLM API enabled.                                               |
| OpenWeatherMap API key is required for weather data retrieval.                                             | https://openweathermap.org/api                                                                       |
| NewsAPI key is required to fetch news headlines.                                                          | https://newsapi.org/                                                                                |
| Google Calendar OAuth2 credentials must be set up with access to the user's calendar.                       | https://developers.google.com/calendar/api                                                          |
| Telegram Bot API token and chat ID needed to send audio messages.                                          | https://core.telegram.org/bots/api                                                                   |
| ffmpeg must be installed on the host running n8n to convert PCM audio files to MP3.                        | https://ffmpeg.org/download.html                                                                    |
| The workflow is designed as a daily briefing podcast generator but can be adapted for other personalized AI briefings. |                                                                                                    |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.