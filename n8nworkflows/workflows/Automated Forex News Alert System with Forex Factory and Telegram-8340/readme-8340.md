Automated Forex News Alert System with Forex Factory and Telegram

https://n8nworkflows.xyz/workflows/automated-forex-news-alert-system-with-forex-factory-and-telegram-8340


# Automated Forex News Alert System with Forex Factory and Telegram

---
### 1. Workflow Overview

This workflow automates Forex news alerts by integrating Forex Factory calendar events with Telegram messaging. It monitors Forex Factory economic calendar events via Google Calendar, extracts key data such as actual and forecast values of economic indicators, interprets their significance relative to the currency involved, and sends conditional alerts to a Telegram chat.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Triggering on Forex Factory events via Google Calendar.
- **1.2 News Detail Extraction:** Parsing and structuring event details including dates, currency codes, forecast, and previous values.
- **1.3 News Content Scraping:** Using a web scraper to retrieve detailed economic data from the event’s news link.
- **1.4 Data Validation and Conversion:** Parsing actual and forecast values, handling special suffixes (%KMBT), and converting strings to numbers.
- **1.5 Decision Logic:** Determining whether the data indicates a positive or negative impact on the currency.
- **1.6 Notification Dispatch:** Sending formatted messages to Telegram based on the analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for Forex Factory economic calendar events starting, using a Google Calendar trigger node.

- **Nodes Involved:**  
  - Google Calendar Trigger  
  - IF Has Forecast  
  - No Operation, do nothing1 (for events without forecast)

- **Node Details:**  
  - **Google Calendar Trigger**  
    - *Type:* Google Calendar Trigger  
    - *Role:* Starts the workflow when a calendar event starts.  
    - *Configuration:* Polls every minute; triggers on "eventStarted" from a specified Forex Factory calendar.  
    - *Credentials:* Google OAuth2 (configured with appropriate access).  
    - *Edge Cases:* Calendar ID or OAuth credentials misconfiguration leads to trigger failure. Polling frequency could cause delays or API quota issues.

  - **IF Has Forecast**  
    - *Type:* If node  
    - *Role:* Filters events to only those containing the text "Forecast: " in their description.  
    - *Expression:* Checks if `$json.description` contains "Forecast: ".  
    - *Input:* From the Google Calendar Trigger.  
    - *Output:*  
      - If true: proceeds to "Get News Details".  
      - If false: proceeds to "No Operation, do nothing1".  
    - *Edge Cases:* Events with unusual formatting may bypass detection.  

  - **No Operation, do nothing1**  
    - *Type:* No Operation (NoOp)  
    - *Role:* Ends workflow branches where forecast data is missing.  
    - *Input:* From "IF Has Forecast" false branch.

---

#### 2.2 News Detail Extraction

- **Overview:**  
  Extracts structured details from the calendar event, such as the news link, forecast, previous values, date components, currency codes, and impact level.

- **Nodes Involved:**  
  - Get News Details

- **Node Details:**  
  - **Get News Details**  
    - *Type:* Set node  
    - *Role:* Parses multiple fields from the event JSON, using string operations and conditional logic to map month numbers to abbreviations, extract URLs, and identify currency codes.  
    - *Key Expressions:*  
      - Extracts URL from the description using `.extractUrl()`.  
      - Splits description text by double line breaks and spaces to parse forecast and previous values.  
      - Maps month number to 3-letter month abbreviation via nested `$if` expressions.  
      - Converts date parts (year, month, day) from ISO string.  
      - Determines currency code by mapping summary abbreviations (e.g., "US" → "USD", "AU" → "AUD", etc.).  
      - Extracts impact string from description.  
    - *Input:* From "IF Has Forecast" true branch.  
    - *Output:* To "Wait 10s".  
    - *Edge Cases:*  
      - Date parsing assumes valid ISO format in event start time.  
      - Currency mapping defaults to "XXX" if unknown.  
      - Unexpected description formats may cause incorrect parsing.  

---

#### 2.3 News Content Scraping

- **Overview:**  
  Waits briefly, then scrapes the actual news data from the extracted news link, retrieving updated actual and forecast values.

- **Nodes Involved:**  
  - Wait 10s  
  - Scrape News Link  
  - Get Actual Data?  
  - Actual, Forecast Value  
  - Wait 5s (alternative path)

- **Node Details:**  
  - **Wait 10s**  
    - *Type:* Wait node  
    - *Role:* Delays execution by 10 seconds to allow the source page to update if necessary.  
    - *Input:* From "Get News Details".  
    - *Output:* To "Scrape News Link".

  - **Scrape News Link**  
    - *Type:* Airtop (web scraper) node  
    - *Role:* Scrapes the HTML content of the news link to extract tabular economic data.  
    - *Configuration:* Uses a new session per request with the URL from "newsLink".  
    - *Credentials:* Airtop API credentials required.  
    - *Input:* From "Wait 10s".  
    - *Output:* To "Get Actual Data?".  
    - *Edge Cases:* Scraper failures due to network, site changes, or API quota limits.

  - **Wait 5s**  
    - *Type:* Wait node  
    - *Role:* Alternative wait node used in an error branch or retry scenario.  
    - *Input:* From "Get Actual Data?" false branch.  
    - *Output:* To "Scrape News Link".

  - **Get Actual Data?**  
    - *Type:* If node  
    - *Role:* Checks if the scraped content includes the correct date matching the news details.  
    - *Logic:*  
      - Extracts a substring from scraped content and checks if it contains the news date or adjusted dates (day - 1, month - 1 with days 28-31).  
      - Multiple date checks cover edge cases for month-end overlaps.  
    - *Input:* From "Scrape News Link".  
    - *Output:*  
      - True: proceeds to "Actual, Forecast Value".  
      - False: loops back via "Wait 5s" to "Scrape News Link" (retry).  
    - *Edge Cases:* Date mismatch could cause indefinite loops; the retry wait mitigates this.

  - **Actual, Forecast Value**  
    - *Type:* Set node  
    - *Role:* Extracts 'actual' and 'forecast' values from the scraped content using substring and split operations; cleans escaped quotes.  
    - *Also sets:* currency code from "Get News Details".  
    - *Input:* From "Get Actual Data?" true branch.  
    - *Output:* To "If" node (next block).  
    - *Edge Cases:* Parsing failures if scraped content format changes.

---

#### 2.4 Data Validation and Conversion

- **Overview:**  
  Processes actual and forecast values to remove suffixes like %, K, M, B, T and converts strings to numeric types.

- **Nodes Involved:**  
  - If  
  - Delete %KMBT, To Number  
  - To Number

- **Node Details:**  
  - **If**  
    - *Type:* If node  
    - *Role:* Checks if 'forecast' value contains any of the suffixes "%", "K", "M", "B", or "T".  
    - *Input:* From "Actual, Forecast Value".  
    - *Output:*  
      - True: goes to "Delete %KMBT, To Number".  
      - False: goes to "To Number".  
    - *Edge Cases:* Ensures correct numeric conversion path is selected.

  - **Delete %KMBT, To Number**  
    - *Type:* Set node  
    - *Role:* Strips the last character (assumed suffix) from 'actual' and 'forecast' strings, then converts to number.  
    - *Input:* From "If" true branch.  
    - *Output:* To "'Actual' less than 'Forecast' is good for currency?".  
    - *Edge Cases:* Assumes suffix is exactly one character at the end; multi-character suffixes or malformed data may cause errors.

  - **To Number**  
    - *Type:* Set node  
    - *Role:* Directly converts 'actual' and 'forecast' strings to numbers without stripping suffixes.  
    - *Input:* From "If" false branch.  
    - *Output:* To "'Actual' less than 'Forecast' is good for currency?".  
    - *Edge Cases:* Conversion errors if strings are non-numeric.

---

#### 2.5 Decision Logic

- **Overview:**  
  Determines whether the relationship between actual and forecast values implies a positive or negative impact on the currency, considering the specific interpretation from news content.

- **Nodes Involved:**  
  - 'Actual' less than 'Forecast' is good for currency?  
  - Less Good  
  - Greater Good  
  - No Operation, do nothing

- **Node Details:**  
  - **'Actual' less than 'Forecast' is good for currency?**  
    - *Type:* If node  
    - *Role:* Checks if the scraped news text contains the phrase "'Actual' less than 'Forecast' is good for currency;".  
    - *Input:* From the previous numeric conversion nodes.  
    - *Output:*  
      - True: goes to "Less Good" node (logic inverted).  
      - False: goes to "Greater Good" node (standard logic).  
    - *Edge Cases:* If phrase is missing or altered, decision may be incorrect.

  - **Less Good**  
    - *Type:* Telegram node  
    - *Role:* Sends a Telegram message indicating whether the actual value is good or bad for the currency, assuming the "less than forecast" interpretation.  
    - *Message:* Includes event summary, impact, actual and forecast values, and a conditional statement:  
      - If actual == forecast → "NEUTRAL for [currency]"  
      - If actual < forecast → "GOOD for [currency]"  
      - Else → "BAD for [currency]"  
    - *Credentials:* Telegram API credentials configured.  
    - *Input:* From "'Actual' less than 'Forecast' is good for currency?" true branch.  
    - *Output:* To "No Operation, do nothing".  
    - *Edge Cases:* Telegram API errors, chat ID misconfiguration.

  - **Greater Good**  
    - *Type:* Telegram node  
    - *Role:* Similar to "Less Good" but with inverted logic:  
      - If actual == forecast → "NEUTRAL for [currency]"  
      - If actual > forecast → "GOOD for [currency]"  
      - Else → "BAD for [currency]"  
    - *Credentials:* Telegram API credentials.  
    - *Input:* From "'Actual' less than 'Forecast' is good for currency?" false branch.  
    - *Output:* To "No Operation, do nothing".  
    - *Edge Cases:* Same as "Less Good".

  - **No Operation, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Terminates successful message sending branch.

---

### 3. Summary Table

| Node Name                                       | Node Type                  | Functional Role                          | Input Node(s)                         | Output Node(s)                         | Sticky Note |
|------------------------------------------------|----------------------------|----------------------------------------|-------------------------------------|---------------------------------------|-------------|
| Google Calendar Trigger                         | Google Calendar Trigger    | Trigger workflow on Forex Factory event start | -                                   | IF Has Forecast                       |             |
| IF Has Forecast                                | If                         | Filter events containing forecast info | Google Calendar Trigger             | Get News Details, No Operation, do nothing1 |             |
| No Operation, do nothing1                       | No Operation               | Ends branch for events without forecast | IF Has Forecast                    | -                                     |             |
| Get News Details                               | Set                        | Extract news URL, dates, currency, impact | IF Has Forecast                   | Wait 10s                             |             |
| Wait 10s                                       | Wait                       | Delay before scraping news content    | Get News Details                   | Scrape News Link                     |             |
| Scrape News Link                               | Airtop Scraper             | Scrape economic data from news link   | Wait 10s                          | Get Actual Data?                     |             |
| Get Actual Data?                               | If                         | Check if scraped data matches event date | Scrape News Link                | Actual, Forecast Value, Wait 5s      |             |
| Wait 5s                                        | Wait                       | Wait before retrying scraping          | Get Actual Data? (false branch)   | Scrape News Link                     |             |
| Actual, Forecast Value                         | Set                        | Extract actual and forecast values from scraped data | Get Actual Data? (true branch) | If                                  |             |
| If                                             | If                         | Check for suffixes in forecast value   | Actual, Forecast Value            | Delete %KMBT, To Number / To Number |             |
| Delete %KMBT, To Number                        | Set                        | Remove suffixes and convert to number  | If (true branch)                  | 'Actual' less than 'Forecast' is good for currency? |             |
| To Number                                      | Set                        | Convert values to number directly       | If (false branch)                 | 'Actual' less than 'Forecast' is good for currency? |             |
| 'Actual' less than 'Forecast' is good for currency? | If                         | Check interpretation of impact direction | Delete %KMBT, To Number / To Number | Less Good / Greater Good             |             |
| Less Good                                      | Telegram                   | Send Telegram alert for "less is good" interpretation | 'Actual' less than 'Forecast' is good for currency? (true branch) | No Operation, do nothing            |             |
| Greater Good                                   | Telegram                   | Send Telegram alert for "greater is good" interpretation | 'Actual' less than 'Forecast' is good for currency? (false branch) | No Operation, do nothing            |             |
| No Operation, do nothing                       | No Operation               | End of alert sending branch             | Less Good, Greater Good           | -                                     |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Calendar Trigger Node:**
   - Set to trigger on event start.
   - Poll every minute.
   - Configure with Google OAuth2 credentials.
   - Set calendarId to the Forex Factory calendar.

2. **Add an If Node "IF Has Forecast":**
   - Condition: `$json.description` contains "Forecast: ".
   - Connect output true to next step; false to a NoOp node.

3. **Add a No Operation Node "No Operation, do nothing1":**
   - Connect input from "IF Has Forecast" false branch.

4. **Add a Set Node "Get News Details":**
   - Parse event description to extract:
     - newsLink (extract URL from description)
     - forecast (parse from description text)
     - previous (parse from description)
     - year (first 4 chars of start.dateTime)
     - month (map month number to abbreviation with nested `$if`)
     - month-1 (month before current using same mapping)
     - day (day number from start.dateTime)
     - day-1 (day minus 1 with fallback to 30)
     - currency (map summary country code to Forex code with nested `$if`)
     - impact (parse from description)
   - Connect input from "IF Has Forecast".

5. **Add a Wait Node "Wait 10s":**
   - Delay 10 seconds.
   - Connect input from "Get News Details".

6. **Add Airtop Scraper Node "Scrape News Link":**
   - Set URL from `newsLink`.
   - Use new session mode.
   - Configure Airtop API credentials.
   - Connect input from "Wait 10s".

7. **Add an If Node "Get Actual Data?":**
   - Check if scraped content text substring contains event date or adjusted dates (day-1, month-1 with days 28-31).
   - Connect input from "Scrape News Link".
   - True branch connects to "Actual, Forecast Value".
   - False branch connects to "Wait 5s".

8. **Add a Wait Node "Wait 5s":**
   - Delay 5 seconds.
   - Connect input from "Get Actual Data?" false branch.
   - Connect output back to "Scrape News Link" (loop retry).

9. **Add a Set Node "Actual, Forecast Value":**
   - Extract 'actual' and 'forecast' by substring and split of scraped content.
   - Clean escaped quotes if present.
   - Assign 'currency' from "Get News Details".
   - Connect input from "Get Actual Data?" true branch.

10. **Add an If Node "If":**
    - Check if 'forecast' contains any of "%", "K", "M", "B", or "T".
    - Connect input from "Actual, Forecast Value".
    - True branch to "Delete %KMBT, To Number".
    - False branch to "To Number".

11. **Add a Set Node "Delete %KMBT, To Number":**
    - Remove last character from 'actual' and 'forecast'.
    - Convert both to numbers.
    - Connect input from "If" true branch.

12. **Add a Set Node "To Number":**
    - Convert 'actual' and 'forecast' strings directly to numbers.
    - Connect input from "If" false branch.

13. **Add an If Node "'Actual' less than 'Forecast' is good for currency?":**
    - Check if scraped text contains "'Actual' less than 'Forecast' is good for currency;".
    - Connect input from both numeric conversion nodes.
    - True branch to "Less Good".
    - False branch to "Greater Good".

14. **Add a Telegram Node "Less Good":**
    - Send message to Telegram chat with:
      ```
      [Event summary]
      Impact: [impact]
      Actual: [actual]
      Forecast: [forecast]
      [Conditional text]
      ```
      Conditional logic:
      - If actual == forecast → "NEUTRAL for [currency]"
      - If actual < forecast → "GOOD for [currency]"
      - Else → "BAD for [currency]"
    - Configure Telegram credentials and chat ID.
    - Connect input from "'Actual' less than 'Forecast' is good for currency?" true branch.

15. **Add a Telegram Node "Greater Good":**
    - Same as "Less Good" but with conditional:
      - If actual == forecast → "NEUTRAL for [currency]"
      - If actual > forecast → "GOOD for [currency]"
      - Else → "BAD for [currency]"
    - Configure Telegram credentials and chat ID.
    - Connect input from "'Actual' less than 'Forecast' is good for currency?" false branch.

16. **Add a No Operation Node "No Operation, do nothing":**
    - Connect input from both Telegram nodes to end workflow branches.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow depends on a Google Calendar feed from Forex Factory's economic calendar.                    | Forex Factory calendar integration.                                                            |
| Airtop API credentials are required to scrape news content reliably.                                      | Airtop web scraping service: https://airtop.io/                                                |
| Telegram API credentials must be set up with a valid bot token and chat ID for notification delivery.     | Telegram Bot API documentation: https://core.telegram.org/bots/api                              |
| The workflow uses nested conditional expressions extensively to parse dates and convert currencies.       | Date parsing and currency mapping is sensitive to event data format changes.                    |
| The polling interval is set to every minute; adjust accordingly based on API rate limits and requirements. | Google Calendar API quota considerations.                                                      |
| Potential infinite loops mitigated by wait nodes and date validation in "Get Actual Data?" node.           | Be mindful to monitor logs for excessive retries.                                              |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow constructed with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and publicly accessible.