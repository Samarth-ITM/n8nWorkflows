Log Daily Mood & Weather to Notion with Telegram and OpenWeatherMap

https://n8nworkflows.xyz/workflows/log-daily-mood---weather-to-notion-with-telegram-and-openweathermap-4862


# Log Daily Mood & Weather to Notion with Telegram and OpenWeatherMap

### 1. Workflow Overview

This n8n workflow automates the daily logging of personal mood and weather data into Notion, leveraging Telegram for mood input and OpenWeatherMap for weather data retrieval. It targets users seeking to track daily emotional states alongside weather conditions for personal insight or journaling purposes.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception**: Initiates daily mood prompt via Telegram and waits for user mood response.
- **1.2 Weather Data Retrieval**: Fetches current weather data based on the user’s city name or latitude/longitude.
- **1.3 Data Logging to Notion**: Retrieves the Notion database and logs mood and weather data as new entries.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block triggers a daily prompt sent to a Telegram user to capture their mood. It listens for the user’s mood response through Telegram triggers.

- **Nodes Involved:**  
  - Daily Trigger  
  - Send Mood Prompt (Telegram)  
  - Wait for Mood Response (Telegram Trigger)  
  - Wait for Mood Response1 (Telegram Trigger - disabled)

- **Node Details:**

  - **Daily Trigger**  
    - Type: Cron Trigger  
    - Role: Fires the workflow daily at a configured time (default cron schedule not detailed here) to initiate mood prompt.  
    - Inputs: None  
    - Outputs: Send Mood Prompt  
    - Failure Cases: Misconfigured cron expression, n8n server downtime.

  - **Send Mood Prompt**  
    - Type: Telegram node (Send Message)  
    - Role: Sends a mood prompt message to the user’s Telegram account.  
    - Key Config: Uses a configured Telegram bot credential with a webhook ID. The message content is set to prompt the user for their mood.  
    - Inputs: Daily Trigger  
    - Outputs: Wait for Mood Response  
    - Failure Cases: Telegram API errors, invalid bot token, network issues.

  - **Wait for Mood Response**  
    - Type: Telegram Trigger  
    - Role: Waits for the user to reply with their mood via Telegram.  
    - Key Config: Uses a webhook to listen for incoming messages on the same Telegram bot.  
    - Inputs: Send Mood Prompt (via workflow continuation)  
    - Outputs: Get Weather using city name  
    - Failure Cases: User does not respond, webhook misconfiguration, Telegram API downtime.

  - **Wait for Mood Response1**  
    - Type: Telegram Trigger  
    - Role: Duplicate of "Wait for Mood Response" but disabled, possibly a backup or alternative trigger setup.  
    - Inputs/Outputs: None active since disabled.

---

#### 1.2 Weather Data Retrieval

- **Overview:**  
Fetches weather data from OpenWeatherMap API based on city name or geographic coordinates obtained from the mood response.

- **Nodes Involved:**  
  - Get Weather using city name (HTTP Request)  
  - Get Weather using lat/lon (HTTP Request)  
  - Retrieve City Weather (Code)  
  - Retrieve City Weather1 (Code)

- **Node Details:**

  - **Get Weather using city name**  
    - Type: HTTP Request  
    - Role: Queries OpenWeatherMap API for weather data using the city name extracted from the mood response.  
    - Configuration: HTTP GET request with dynamic URL parameters including city name and API key (stored in credentials).  
    - Input: Wait for Mood Response  
    - Output: Retrieve City Weather  
    - Failure Cases: Invalid city name, API key issues, network errors, API rate limits.

  - **Get Weather using lat/lon**  
    - Type: HTTP Request  
    - Role: Queries OpenWeatherMap API using geographic coordinates (latitude and longitude).  
    - Input: Wait for Mood Response1 (disabled path, likely inactive)  
    - Output: Retrieve City Weather1  
    - Failure Cases: Invalid coordinates, API or network errors.

  - **Retrieve City Weather**  
    - Type: Code (JavaScript)  
    - Role: Processes or formats the weather data retrieved from HTTP Request before sending it to Notion. May extract specific fields or prepare data.  
    - Input: Get Weather using city name  
    - Output: Retrieve Database  
    - Failure Cases: Code errors, undefined fields, JSON parsing errors.

  - **Retrieve City Weather1**  
    - Type: Code (JavaScript)  
    - Role: Similar to Retrieve City Weather, but for the lat/lon weather path.  
    - Input: Get Weather using lat/lon  
    - Output: Retrieve Database1  
    - Failure Cases: Same as above.

---

#### 1.3 Data Logging to Notion

- **Overview:**  
Retrieves the appropriate Notion database and adds new rows containing the combined mood and weather data.

- **Nodes Involved:**  
  - Retrieve Database (Notion)  
  - Add row into Notion (Notion)  
  - Retrieve Database1 (Notion)  
  - Add row into Notion1 (Notion)

- **Node Details:**

  - **Retrieve Database**  
    - Type: Notion node (Database Query)  
    - Role: Fetches metadata or schema from the Notion database used for logging mood and weather data (city name path).  
    - Input: Retrieve City Weather  
    - Output: Add row into Notion  
    - Failure Cases: Credential issues with Notion OAuth, database ID invalid, API limits.

  - **Add row into Notion**  
    - Type: Notion node (Create Page)  
    - Role: Inserts a new entry into the Notion database combining mood and weather details.  
    - Input: Retrieve Database  
    - Output: None  
    - Failure Cases: Same as Retrieve Database, plus invalid data format.

  - **Retrieve Database1**  
    - Type: Notion node (Database Query)  
    - Role: Same as Retrieve Database but used for lat/lon weather path.  
    - Input: Retrieve City Weather1  
    - Output: Add row into Notion1  
    - Failure Cases: Same as above.

  - **Add row into Notion1**  
    - Type: Notion node (Create Page)  
    - Role: Adds new entry to Notion database for lat/lon weather data.  
    - Input: Retrieve Database1  
    - Output: None  
    - Failure Cases: Same as Add row into Notion.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                           | Input Node(s)          | Output Node(s)           | Sticky Note |
|------------------------|----------------------|-----------------------------------------|------------------------|--------------------------|-------------|
| Daily Trigger          | Cron Trigger         | Initiate workflow daily                  |                        | Send Mood Prompt          |             |
| Send Mood Prompt       | Telegram             | Send daily mood prompt to user           | Daily Trigger          | Wait for Mood Response    |             |
| Wait for Mood Response | Telegram Trigger     | Wait for user mood reply                  | Send Mood Prompt       | Get Weather using city name |             |
| Wait for Mood Response1| Telegram Trigger (disabled) | Alternate mood response listener (disabled) |                      | Get Weather using lat/lon |             |
| Get Weather using city name | HTTP Request     | Fetch weather by city name                | Wait for Mood Response | Retrieve City Weather     |             |
| Get Weather using lat/lon | HTTP Request       | Fetch weather by geographic coordinates   | Wait for Mood Response1 (disabled) | Retrieve City Weather1 |             |
| Retrieve City Weather  | Code (JavaScript)    | Process weather data (city name)          | Get Weather using city name | Retrieve Database       |             |
| Retrieve City Weather1 | Code (JavaScript)    | Process weather data (lat/lon)            | Get Weather using lat/lon | Retrieve Database1      |             |
| Retrieve Database      | Notion               | Retrieve Notion database info (city name) | Retrieve City Weather  | Add row into Notion       |             |
| Add row into Notion    | Notion               | Add mood and weather entry (city name)   | Retrieve Database      |                          |             |
| Retrieve Database1     | Notion               | Retrieve Notion database info (lat/lon)  | Retrieve City Weather1 | Add row into Notion1      |             |
| Add row into Notion1   | Notion               | Add mood and weather entry (lat/lon)     | Retrieve Database1     |                          |             |
| Sticky Note            | Sticky Note          | Various notes (empty)                      |                        |                          |             |
| Sticky Note1           | Sticky Note          | Various notes (empty)                      |                        |                          |             |
| Sticky Note2           | Sticky Note          | Various notes (empty)                      |                        |                          |             |
| Sticky Note3           | Sticky Note          | Various notes (empty)                      |                        |                          |             |
| Sticky Note4           | Sticky Note          | Various notes (empty)                      |                        |                          |             |
| Sticky Note5           | Sticky Note          | Various notes (empty)                      |                        |                          |             |
| Sticky Note6           | Sticky Note          | Various notes (empty)                      |                        |                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node** named `Daily Trigger`.  
   - Set it to trigger daily at the desired time (e.g., every day at 8 AM).

2. **Add a Telegram node** named `Send Mood Prompt`.  
   - Configure Telegram credentials (OAuth2 bot token).  
   - Set the message text prompting the user to submit their mood.  
   - Connect `Daily Trigger` output to `Send Mood Prompt` input.

3. **Add a Telegram Trigger node** named `Wait for Mood Response`.  
   - Use the same Telegram bot credentials and webhook ID as in `Send Mood Prompt`.  
   - Configure it to listen for incoming messages (user mood).  
   - Connect `Send Mood Prompt` output to `Wait for Mood Response` input (for logical flow).

4. **Add HTTP Request node** named `Get Weather using city name`.  
   - Configure it to call OpenWeatherMap API with a GET request.  
   - Use parameters: city name extracted from Telegram message, API key stored in credentials.  
   - Connect `Wait for Mood Response` output to this node.

5. **Add a Code node** named `Retrieve City Weather`.  
   - Write JavaScript code to parse and reformat the weather API response, extracting relevant data fields (e.g., temperature, description).  
   - Connect `Get Weather using city name` output to this node.

6. **Add Notion node** named `Retrieve Database`.  
   - Configure with Notion OAuth2 credentials.  
   - Set to retrieve the target Notion database (use database ID).  
   - Connect `Retrieve City Weather` output to this node.

7. **Add Notion node** named `Add row into Notion`.  
   - Configure to create a new page/row in the database.  
   - Map fields to include mood input from Telegram and processed weather data.  
   - Connect `Retrieve Database` output to this node.

8. **Optionally**, replicate steps 4–7 with the alternative weather path using geographic coordinates (`Get Weather using lat/lon`, `Retrieve City Weather1`, `Retrieve Database1`, `Add row into Notion1`), connected from the disabled Telegram Trigger `Wait for Mood Response1` if you plan to activate that.

9. **Ensure all credentials are configured:**  
   - Telegram Bot credentials with proper webhook URL set and active.  
   - OpenWeatherMap API key configured in HTTP Request nodes.  
   - Notion OAuth2 credentials with database access.

10. **Test the workflow:**  
    - Manually trigger or wait for the scheduled time.  
    - Verify mood prompt is sent via Telegram.  
    - Respond via Telegram to enable weather retrieval and data logging.  
    - Check Notion database for new entries with mood and weather info.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Telegram integration requires setting up a Telegram bot and obtaining a valid OAuth2 token and webhook.  | [Telegram Bot API](https://core.telegram.org/bots/api) |
| OpenWeatherMap API key needed for weather data retrieval; free tier may have limits.                      | [OpenWeatherMap API](https://openweathermap.org/api) |
| Notion integration requires OAuth2 setup and database ID of target database for logging entries.          | [Notion API Documentation](https://developers.notion.com/) |
| Workflow disabled the alternative latitude/longitude weather retrieval path — activate if needed.         | Internal workflow design note                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow, fully compliant with content policies, containing no illegal, offensive, or protected content. All manipulated data is legal and public.