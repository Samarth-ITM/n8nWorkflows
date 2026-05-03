Track Forex News Releases with MyFxBook Data & Send Alerts to Telegram & Google Sheets

https://n8nworkflows.xyz/workflows/track-forex-news-releases-with-myfxbook-data---send-alerts-to-telegram---google-sheets-8491


# Track Forex News Releases with MyFxBook Data & Send Alerts to Telegram & Google Sheets

### 1. Workflow Overview

This workflow is designed to track Forex news releases from Google Calendar events (typically linked to Forex Factory news), scrape live Forex data from MyFxBook, and send alerts to Telegram while logging relevant data to Google Sheets. It targets Forex traders and analysts who want timely notifications and historic record-keeping of Forex news impact on currency pairs.

**Use cases:**
- Receive alerts on Forex news releases with actual vs. forecast data.
- Determine whether the news is positive or negative for a currency and notify via Telegram.
- Log detailed news and price data for further analysis in Google Sheets.
- Use news alerts as triggers for trading actions (e.g., MetaTrader 4).

The workflow is logically organized into these blocks:

- **1.1 Event Trigger & Initial Filtering:** Trigger on news release events from Google Calendar; filter events with forecast data.
- **1.2 News Details Extraction:** Extract news metadata including currency, impact, date, and news URL.
- **1.3 Wait & Scrape Actual Data:** Pause to ensure actual news data is published, then scrape it from the news link.
- **1.4 Actual vs Forecast Processing:** Extract, clean, convert, and compare actual and forecast values.
- **1.5 Sentiment Analysis & Telegram Alerts:** Decide if news impact is positive or negative based on comparison and predefined logic; send Telegram notifications.
- **1.6 Currency Pair Handling & MyFxBook Price Fetch:** Identify affected currency pairs, get live prices from MyFxBook for each pair.
- **1.7 Data Logging to Google Sheets:** Append all relevant data (news, impact, price, position) to Google Sheets.
- **1.8 Control Flows & No-Op Nodes:** Manage flow control for cases where no action is needed.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Event Trigger & Initial Filtering

**Overview:**  
Receives calendar events from Google Calendar triggering on event start; filters only events that contain numerical forecast values.

**Nodes Involved:**  
- Google Calendar Trigger  
- IF Has Forecast  
- No Operation, do nothing1

**Node Details:**  

- **Google Calendar Trigger**  
  - Type: Trigger  
  - Role: Listens for events starting in a chosen Google Calendar.  
  - Config: Polls every minute. Calendar ID is dynamically selected from a list.  
  - Edge cases: Missing calendar access/auth errors; no events; empty calendarId config can cause failure.  

- **IF Has Forecast**  
  - Type: Conditional branch  
  - Role: Checks if event description contains "Forecast: " indicating a valid numerical forecast.  
  - Config: String contains operation on event description field.  
  - Output: If yes, passes to "Get News Details"; else, to No Operation.  
  - Edge cases: Misformatted descriptions may cause false negatives.  

- **No Operation, do nothing1**  
  - Type: NoOp  
  - Role: Stops workflow silently for events without forecast.  

---

#### 1.2 News Details Extraction

**Overview:**  
Extracts structured details from the event description and start time, including news link, forecast, previous value, date parts, currency code, and impact level.

**Nodes Involved:**  
- Get News Details  
- Affected Pairs

**Node Details:**  

- **Get News Details**  
  - Type: Set  
  - Role: Parses event description string and start time to extract:  
    - newsLink (URL extracted from description)  
    - forecast (numerical string)  
    - previous (previous period value)  
    - year, month, month-1, day, day-1 (date components with string month names)  
    - currency (mapped from event summary keywords like US, AU, etc.)  
    - impact (impact level from description)  
  - Config: Uses substring, split, and conditional expressions to extract and format data.  
  - Edge cases: Event description format changes will break parsing; date edge cases handled with fallback for day-1 and month-1.  

- **Affected Pairs**  
  - Type: Set  
  - Role: Assigns a string of affected currency pairs based on currency extracted. For USD uses all major pairs; for others, selects the relevant pair(s).  
  - Edge cases: Defaults to empty if currency unknown; partial currency abbreviation mapping.  

---

#### 1.3 Wait & Scrape Actual Data

**Overview:**  
Waits a short time after event start to allow actual data publishing, then scrapes actual news data from the news link using Airtop API.

**Nodes Involved:**  
- Wait (10 seconds)  
- Scrape News Link  
- Get Actual Data?  
- Wait 5s (retry wait)

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pause for 10 seconds before scraping to ensure actual data is available.  

- **Scrape News Link**  
  - Type: HTTP Scraper (Airtop node)  
  - Role: Scrapes the news page content at the extracted newsLink URL.  
  - Config: Uses new session mode; extraction operation.  
  - Edge cases: Network errors, scraping failures, or content changes.  

- **Get Actual Data?**  
  - Type: If  
  - Role: Checks if scraped content contains the date corresponding to the news release to confirm actual data presence.  
  - Config: Checks multiple date formats (current day, previous day, last days of previous month) inside scraped text.  
  - Edge cases: Missing or delayed data causes false negatives.  

- **Wait 5s**  
  - Type: Wait  
  - Role: If actual data not found, waits 5 seconds and retries scraping to allow for data availability lag.  

---

#### 1.4 Actual vs Forecast Processing

**Overview:**  
Extracts actual and forecast values from scraped content, removes non-numeric suffixes, converts to numbers, and determines if a lower actual is beneficial for the currency.

**Nodes Involved:**  
- Actual, Forecast Value  
- If  
- Delete %KMBT, To Number  
- To Number  
- 'Actual' less than 'Forecast' is good for currency?

**Node Details:**  

- **Actual, Forecast Value**  
  - Type: Set  
  - Role: Parses scraped table text to extract actual and forecast values as strings; retrieves currency and affectedPairs from prior node.  
  - Config: Uses substring and split operations on scraped text; handles escaped characters.  

- **If**  
  - Type: Conditional  
  - Role: Checks if the forecast string contains any non-numeric characters (%) or K, M, B, T suffixes.  
  - Output: If yes, goes to "Delete %KMBT, To Number", else to "To Number".  

- **Delete %KMBT, To Number**  
  - Type: Set  
  - Role: Strips the last character from actual and forecast strings (assumed suffix) and converts to number.  

- **To Number**  
  - Type: Set  
  - Role: Converts actual and forecast strings directly to numbers (when no suffix).  

- **'Actual' less than 'Forecast' is good for currency?**  
  - Type: If  
  - Role: Checks text from scraped data if a lower actual value is positive for the currency. Routes to 'Less Good' or 'Greater Good' Telegram nodes accordingly.  

---

#### 1.5 Sentiment Analysis & Telegram Alerts

**Overview:**  
Sends Telegram messages based on the comparison results indicating if news impact is good or bad for the currency.

**Nodes Involved:**  
- Less Good (Telegram)  
- Greater Good (Telegram)  
- News Result

**Node Details:**  

- **Less Good**  
  - Type: Telegram  
  - Role: Sends a message to Telegram chat with news summary, impact, actual and forecast values, and a judgment (NEUTRAL/GOOD/BAD) based on actual < forecast logic for "less is good" currencies.  
  - Config: Uses expressions combining data from multiple nodes.  
  - Edge cases: Telegram API errors, invalid chat ID or bot token.  

- **Greater Good**  
  - Type: Telegram  
  - Role: Similar to Less Good but assumes "greater is good" for currency; logic reversed.  

- **News Result**  
  - Type: Set  
  - Role: Sets a simple "GOOD" or "BAD" string based on if the Telegram message contains "GOOD". Also passes affectedPairs array forward.  

---

#### 1.6 Currency Pair Handling & MyFxBook Price Fetch

**Overview:**  
Splits affected currency pairs, checks conditions based on whether the news impact is good or bad, fetches live price data for each pair from MyFxBook.

**Nodes Involved:**  
- Split Out  
- Base=NewsCurrency, Good  
- Base=NewsCurrency, Bad  
- Quote=NewsCurrency, Good  
- Quote=NewsCurrency, Bad  
- BUY (Set)  
- SELL (Set)  
- MyFxBook  
- MyFxBook2  
- Get Price  
- Get Price2

**Node Details:**  

- **Split Out**  
  - Type: SplitOut  
  - Role: Splits the affectedPairs string by comma to process each pair individually.  

- **Base=NewsCurrency, Good / Bad**  
  - Type: If  
  - Role: Filters pairs where the base currency matches the news currency and the news result is good or bad, respectively.  

- **Quote=NewsCurrency, Good / Bad**  
  - Type: If  
  - Role: Filters pairs where the quote currency matches the news currency and news result is good or bad, respectively.  

- **BUY / SELL**  
  - Type: Set  
  - Role: Sets the position action ("BUY" or "SELL") along with date, news, impact, and other relevant data for logging and notification.  
  - BUY is sent for base currency good or quote currency bad; SELL for base currency bad or quote currency good.  

- **MyFxBook / MyFxBook2**  
  - Type: HTTP Request  
  - Role: Requests historical data page from MyFxBook for the currency pair.  
  - Config: URL constructed dynamically with affectedPairs string.  
  - Edge cases: Network errors, 404 if pair unknown, page format changes.  

- **Get Price / Get Price2**  
  - Type: Set  
  - Role: Extracts the current price from the MyFxBook page HTML by parsing the 'data-value' attribute. Converts to number.  

---

#### 1.7 Data Logging to Google Sheets

**Overview:**  
Appends the compiled news and price data with position info into a Google Sheets document for record-keeping and analysis.

**Nodes Involved:**  
- Input to Sheets  
- Input to Sheets2  
- No Operation, do nothing

**Node Details:**  

- **Input to Sheets / Input to Sheets2**  
  - Type: Google Sheets Append Rows  
  - Role: Inserts a new row with columns DATE, NEWS, PAIRS, PRICE, IMPACT, POSITION, and additional optional columns.  
  - Config: Document ID and Sheet name are predefined; columns mapped from prior nodes.  
  - Edge cases: Google API auth failures, quota limits, sheet access permissions.  

- **No Operation, do nothing**  
  - Role: Null node to terminate flow after data insertion or in fallback cases.  

---

#### 1.8 Control Flows & No-Op Nodes

**Overview:**  
Includes no-operation nodes to gracefully end branches where no further action is required, preventing workflow errors or indefinite runs.

**Nodes Involved:**  
- No Operation, do nothing  
- No Operation, do nothing1  
- No Operation, do nothing (multiple instances)

---

### 3. Summary Table

| Node Name                          | Node Type                    | Functional Role                                    | Input Node(s)                  | Output Node(s)                            | Sticky Note                                                                                                                                    |
|-----------------------------------|------------------------------|---------------------------------------------------|-------------------------------|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Google Calendar Trigger            | Google Calendar Trigger       | Trigger on news event start                        |                               | IF Has Forecast                           | ## Get Forex Factory News Release to Telegram, Google Sheets. Record News Data and Live Price from MyFxBook for Affected Currency Pairs. ...     |
| IF Has Forecast                   | If                           | Filter events containing numerical forecast       | Google Calendar Trigger        | Get News Details, No Operation, do nothing1 |                                                                                                                                               |
| No Operation, do nothing1          | NoOp                         | Stops processing for non-forecast events          | IF Has Forecast               |                                           |                                                                                                                                               |
| Get News Details                  | Set                          | Extract news metadata (link, date, currency, etc.)| IF Has Forecast               | Affected Pairs                            |                                                                                                                                               |
| Affected Pairs                   | Set                          | Define affected currency pairs based on currency  | Get News Details              | Wait                                     |                                                                                                                                               |
| Wait                             | Wait                         | Pause 10 seconds before scraping                   | Affected Pairs                | Scrape News Link                          |                                                                                                                                               |
| Scrape News Link                 | Airtop (Scraper)             | Scrape actual data from news URL                   | Wait, Wait 5s                 | Get Actual Data?                          |                                                                                                                                               |
| Get Actual Data?                 | If                           | Check if actual data is present in scraped content| Scrape News Link              | Actual, Forecast Value; Wait 5s           |                                                                                                                                               |
| Wait 5s                         | Wait                         | Wait 5 seconds before retrying scrape              | Get Actual Data?              | Scrape News Link                          |                                                                                                                                               |
| Actual, Forecast Value           | Set                          | Extract actual, forecast, currency, affectedPairs | Get Actual Data?              | If                                       |                                                                                                                                               |
| If                              | If                           | Check if forecast contains % or K, M, B, T suffix | Actual, Forecast Value        | Delete %KMBT, To Number; To Number        |                                                                                                                                               |
| Delete %KMBT, To Number          | Set                          | Strip suffix and convert actual & forecast to number | If                         | 'Actual' less than 'Forecast' is good for currency? |                                                                                                                                               |
| To Number                       | Set                          | Convert actual & forecast strings to numbers       | If                           | 'Actual' less than 'Forecast' is good for currency? |                                                                                                                                               |
| 'Actual' less than 'Forecast' is good for currency? | If           | Determine if lower actual is good for currency      | Delete %KMBT, To Number; To Number | Less Good; Greater Good                    |                                                                                                                                               |
| Less Good                      | Telegram                     | Send Telegram message when "less is good" condition | 'Actual' less than 'Forecast' is good for currency? | News Result                             |                                                                                                                                               |
| Greater Good                   | Telegram                     | Send Telegram message when "greater is good" condition | 'Actual' less than 'Forecast' is good for currency? | News Result                             |                                                                                                                                               |
| News Result                    | Set                          | Classify news result as GOOD or BAD                  | Less Good; Greater Good       | Split Out                                |                                                                                                                                               |
| Split Out                     | SplitOut                     | Split affectedPairs string into individual pairs    | News Result                  | Base=NewsCurrency Good/Bad; Quote=NewsCurrency Good/Bad |                                                                                                                                               |
| Base=NewsCurrency, Good        | If                           | Filter pairs with base currency = news currency & news GOOD | Split Out                   | BUY; No Operation                          |                                                                                                                                               |
| Base=NewsCurrency, Bad         | If                           | Filter pairs with base currency = news currency & news BAD | Split Out                   | SELL; No Operation                         |                                                                                                                                               |
| Quote=NewsCurrency, Good       | If                           | Filter pairs with quote currency = news currency & news GOOD | Split Out                   | SELL; No Operation                         |                                                                                                                                               |
| Quote=NewsCurrency, Bad        | If                           | Filter pairs with quote currency = news currency & news BAD | Split Out                   | BUY; No Operation                          |                                                                                                                                               |
| BUY                          | Set                          | Set BUY position data for logging                    | Base=NewsCurrency Good; Quote=NewsCurrency Bad | MyFxBook                               |                                                                                                                                               |
| SELL                         | Set                          | Set SELL position data for logging                   | Base=NewsCurrency Bad; Quote=NewsCurrency Good | MyFxBook2                              |                                                                                                                                               |
| MyFxBook                     | HTTP Request                 | Fetch historical price data for BUY pairs            | BUY                          | Get Price                                |                                                                                                                                               |
| MyFxBook2                    | HTTP Request                 | Fetch historical price data for SELL pairs           | SELL                         | Get Price2                               |                                                                                                                                               |
| Get Price                    | Set                          | Extract current price from MyFxBook response          | MyFxBook                     | Input to Sheets                          |                                                                                                                                               |
| Get Price2                   | Set                          | Extract current price from MyFxBook response          | MyFxBook2                    | Input to Sheets2                         |                                                                                                                                               |
| Input to Sheets              | Google Sheets                | Append BUY data row to Google Sheets                  | Get Price                    | No Operation                            |                                                                                                                                               |
| Input to Sheets2             | Google Sheets                | Append SELL data row to Google Sheets                 | Get Price2                   | No Operation                            |                                                                                                                                               |
| No Operation, do nothing     | NoOp                        | End branch with no action                              | Input to Sheets; Input to Sheets2; Base and Quote currency filters |                                           |                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Calendar Trigger node:**
   - Set to trigger on event start.
   - Poll mode: Every minute.
   - Select your Google Calendar ID (Forex news calendar).
   - Connect output to the next node.

2. **Add an IF node (IF Has Forecast):**
   - Condition: Check if event description contains "Forecast: ".
   - True branch connects to "Get News Details".
   - False branch connects to a NoOp node to stop processing.

3. **Add a No Operation node (No Operation, do nothing1):**
   - Connect false output of IF Has Forecast here.

4. **Add a Set node (Get News Details):**
   - Extract and assign:
     - newsLink: Extract URL from event description.
     - forecast: Extract forecast value from description text.
     - previous: Extract previous value.
     - year: Extract from event start dateTime.
     - month: Convert month number to string month name.
     - month-1: Previous month string.
     - day: Extract day number.
     - day-1: Day minus one, with fallback 30 if zero.
     - currency: Map currency code from event summary (e.g., US → USD).
     - impact: Extract impact level from description.
   - Connect next to "Affected Pairs".

5. **Add a Set node (Affected Pairs):**
   - Assign affectedPairs string based on the currency:
     - USD maps to all major pairs involving USD.
     - Other currencies map to their respective pairs.
   - Connect to a Wait node.

6. **Add a Wait node (Wait 10s):**
   - Wait 10 seconds.
   - Connect to "Scrape News Link".

7. **Add an Airtop node (Scrape News Link):**
   - Set URL to newsLink extracted earlier.
   - Operation: scrape (HTML content).
   - Use new session mode.
   - Connect to "Get Actual Data?".

8. **Add an IF node (Get Actual Data?):**
   - Condition: Check if scraped content contains the news date in various formats (current day, day-1, last days of prior month).
   - True branch connects to "Actual, Forecast Value".
   - False branch connects to a Wait 5s node.

9. **Add a Wait node (Wait 5s):**
   - Wait 5 seconds.
   - Connect back to "Scrape News Link" for retry.

10. **Add a Set node (Actual, Forecast Value):**
    - Parse scraped news content to extract:
      - actual (string)
      - forecast (string)
      - currency (from Get News Details)
      - affectedPairs (from Affected Pairs, split to array)
    - Connect to "If" node.

11. **Add an IF node (If):**
    - Condition: Check if forecast contains non-numeric suffixes (% or K,M,B,T).
    - True branch to "Delete %KMBT, To Number".
    - False branch to "To Number".

12. **Add a Set node (Delete %KMBT, To Number):**
    - Remove last character of actual and forecast strings.
    - Convert both to numbers.
    - Connect to "'Actual' less than 'Forecast' is good for currency?".

13. **Add a Set node (To Number):**
    - Convert actual and forecast strings directly to numbers.
    - Connect to "'Actual' less than 'Forecast' is good for currency?".

14. **Add an IF node ('Actual' less than 'Forecast' is good for currency?):**
    - Condition: Check if scraped content text contains "'Actual' less than 'Forecast' is good for currency;" string.
    - If true, connect to "Less Good" Telegram node.
    - Else, connect to "Greater Good" Telegram node.

15. **Add two Telegram nodes (Less Good & Greater Good):**
    - Compose messages including summary, impact, actual, forecast, and a qualitative judgment (GOOD, BAD, NEUTRAL).
    - Send to your Telegram chat ID using configured bot token.
    - Both nodes connect to "News Result".

16. **Add a Set node (News Result):**
    - Create a newsResult field ("GOOD" if message includes "GOOD", else "BAD").
    - Pass affectedPairs array.
    - Connect to "Split Out".

17. **Add a SplitOut node (Split Out):**
    - Split affectedPairs array into individual currency pairs.
    - Connect outputs to four IF nodes:
      - Base=NewsCurrency, Good
      - Base=NewsCurrency, Bad
      - Quote=NewsCurrency, Good
      - Quote=NewsCurrency, Bad

18. **Add four IF nodes to filter pairs:**
    - For Base=NewsCurrency, check if first 3 characters of pair match currency and newsResult is GOOD or BAD.
    - For Quote=NewsCurrency, check characters 3-6 for match and newsResult is GOOD or BAD.
    - Connect "Good" branches to BUY node, "Bad" branches to SELL node.
    - Connect "else" branches to No Operation nodes.

19. **Add two Set nodes (BUY and SELL):**
    - Assign DATE, NEWS, IMPACT, POSITION ("BUY" or "SELL").
    - Connect BUY to MyFxBook, SELL to MyFxBook2.

20. **Add two HTTP Request nodes (MyFxBook and MyFxBook2):**
    - URL: https://www.myfxbook.com/forex-market/currencies/{{pair}}-historical-data
    - Method: GET
    - Connect to Get Price and Get Price2 respectively.

21. **Add two Set nodes (Get Price and Get Price2):**
    - Extract PRICE from response HTML data using substring and split on 'data-value="'.
    - Convert to number.
    - Connect to Input to Sheets and Input to Sheets2 respectively.

22. **Add two Google Sheets nodes (Input to Sheets and Input to Sheets2):**
    - Operation: Append rows.
    - Document ID: Your Google Sheets ID.
    - Sheet name: gid=0 (or your target sheet).
    - Map columns: DATE, NEWS, PAIRS, PRICE, IMPACT, POSITION.
    - Connect outputs to No Operation nodes.

23. **Add No Operation nodes after sheets input to gracefully end flows.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow captures Forex Factory news releases and sends alerts via Telegram while recording data in Google Sheets. It uses MyFxBook for live price data of affected currency pairs.                                                                                                                                                                                                        | Sticky Note attached to workflow start node.                                                                     |
| The Google Sheets template for logging can be found here: https://docs.google.com/spreadsheets/d/1OhrbUQEc_lGegk5pRWWKz5nrnMbTZGT0lxK9aJqqId4/edit?usp=drive_link                                                                                                                                                                                                                         | Sheets template link from sticky note.                                                                           |
| Required credentials and setup include Google Calendar, Airtop API key, Telegram Bot API token and chat ID, Google Sheets credentials, and enabling Google Drive API in Google Cloud Console.                                                                                                                                                                                               | From workflow description and sticky note.                                                                       |
| Community support available on Discord (https://discord.gg/n8n) and n8n Forum (https://community.n8n.io/).                                                                                                                                                                                                                                                                                   | From sticky note.                                                                                                 |
| Currency mapping in this workflow is partial and may require adjustment depending on additional currencies or changes in news source formatting.                                                                                                                                                                                                                                            | Observed in "Get News Details" node expressions.                                                                  |
| The workflow assumes news event descriptions follow a consistent format to parse forecast, previous, and news links accurately. Changes in source formatting may require node updates.                                                                                                                                                                                                     | Parsing logic in "Get News Details" and "Actual, Forecast Value".                                                 |
| Airtop scraping depends on page structure; if MyFxBook or news pages update their HTML layout, scraping expressions may need adjustment.                                                                                                                                                                                                                                                   | Observed in "Scrape News Link" and price extraction nodes.                                                        |

---

This document should provide a thorough understanding and enable reproduction or modification of the "Track Forex News Releases with MyFxBook Data & Send Alerts to Telegram & Google Sheets" workflow.