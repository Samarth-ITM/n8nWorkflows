📍 Daily Nearby Garage Sales Alerts via Telegram

https://n8nworkflows.xyz/workflows/---daily-nearby-garage-sales-alerts-via-telegram-4649


# 📍 Daily Nearby Garage Sales Alerts via Telegram

### 1. Workflow Overview

This n8n workflow named **"📍 Daily Nearby Garage Sales Alerts via Telegram"** is designed to automatically notify users of local garage sales happening nearby each day via Telegram messages. It targets users interested in receiving timely alerts about nearby garage sales filtered by proximity and event significance.

The workflow is logically divided into the following blocks:

- **1.1 Daily Trigger & Geolocation Setup:** Automatically triggers daily at 7:30 AM (or manually) and fetches the user’s current location from Home Assistant. It then constructs a location-specific URL to retrieve garage sale data.

- **1.2 Fetch & Parse Garage Sale Data:** Performs an HTTP GET request to the dynamically created URL to retrieve raw HTML data from Brocabrac.fr. It extracts the event date and the HTML block containing garage sale events, then checks if events are scheduled for today.

- **1.3 Process, Filter & Extract Relevant Events:** Splits the extracted HTML into individual garage sale entries, loops over each, extracts key details (city, distance, rank), cleans and formats data, and filters events based on distance (≤ 20 km) and presence of rank.

- **1.4 Format & Send Notifications via Telegram:** Formats filtered event data into a user-friendly message string and sends this alert to the user via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Daily Trigger & Geolocation Setup

**Overview:**  
This block ensures the workflow runs automatically every day at a fixed time and determines the user’s current geographic location to tailor data retrieval.

**Nodes Involved:**  
- Every day at 7 AM (Schedule Trigger)  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Get location (Home Assistant)  
- Set URL to parse (Set Node)

**Node Details:**

- **Every day at 7 AM**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow daily at 7:30 AM (cron expression `"30 7 * * *"`).  
  - Inputs: None  
  - Outputs: Connects to "Get location" node  
  - Edge cases: Cron misconfiguration could prevent triggering; time zone issues might affect actual trigger time.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing or debugging.  
  - Inputs: None  
  - Outputs: Connects to "Get location" node  
  - Edge cases: None significant; used only for manual start.

- **Get location**  
  - Type: Home Assistant node  
  - Role: Retrieves user’s current location state from a Home Assistant sensor entity `"Your_Smartphone_location_sensor"`.  
  - Key config: Entity ID set to the smartphone location sensor; fetches the `state` resource.  
  - Inputs: Trigger nodes  
  - Outputs: Connects to "Set URL to parse"  
  - Credentials: Requires valid Home Assistant API credentials.  
  - Edge cases: Sensor unavailable, authentication failure, or stale location data.

- **Set URL to parse**  
  - Type: Set node  
  - Role: Builds a dynamic URL string based on the user’s postal code and locality extracted from the Home Assistant location data.  
  - Key expression:  
    ```
    https://brocabrac.fr/{{ $json.attributes.postal_code.slice(0,2) }}/{{ $json.attributes.locality }}
    ```  
  - Inputs: From "Get location"  
  - Outputs: Connects to "Get Brocabrac" node  
  - Edge cases: Missing or malformed location attributes; string slicing errors.

---

#### 1.2 Fetch & Parse Garage Sale Data

**Overview:**  
This block fetches the HTML page from Brocabrac.fr corresponding to the user’s location and extracts the date and raw HTML content of garage sale listings. It then verifies if any events are scheduled for the current day.

**Nodes Involved:**  
- Get Brocabrac (HTTP Request)  
- Extract Date & Blocks (HTML Extract)  
- Any today? (If node)  
- Extract Garage Sales Events (HTML Extract)  
- Split Out (Split Out)

**Node Details:**

- **Get Brocabrac**  
  - Type: HTTP Request  
  - Role: Fetches the HTML page from the URL set in the previous node.  
  - Config: URL is dynamic, taken from `$json.URL`. Response format is string (raw HTML).  
  - Inputs: "Set URL to parse"  
  - Outputs: "Extract Date & Blocks"  
  - Edge cases: HTTP errors, timeouts, invalid URL, or unexpected HTML structure.

- **Extract Date & Blocks**  
  - Type: HTML Extract  
  - Role: Parses the raw HTML to extract:  
    - `Date`: from the attribute `data-date` of the CSS selector `div.block.ev-list > div:nth-child(1) > div.section-title`  
    - `HTMLBlock`: the full garage sales block HTML from `div.block.ev-list > div`  
  - Inputs: "Get Brocabrac"  
  - Outputs: "Any today?"  
  - Edge cases: HTML structure changes, missing elements, parsing errors.

- **Any today?**  
  - Type: If node  
  - Role: Checks if the extracted `Date` matches the current date (`$today.plus({days})`).  
  - Inputs: "Extract Date & Blocks"  
  - Outputs:  
    - True branch: "Extract Garage Sales Events"  
    - False branch: Ends flow (no events today)  
  - Edge cases: Date formatting errors, timezone mismatches.

- **Extract Garage Sales Events**  
  - Type: HTML Extract  
  - Role: From `HTMLBlock` extracts all individual garage sale event entries matching `div.ev` as an HTML array.  
  - Inputs: "Any today?" (true branch)  
  - Outputs: "Split Out"  
  - Edge cases: No `div.ev` elements found, HTML changes.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of event HTML blocks into individual items for processing.  
  - Field to split out: `ev`  
  - Inputs: "Extract Garage Sales Events"  
  - Outputs: "Loop Over Items"  
  - Edge cases: Empty array, splitting errors.

---

#### 1.3 Process, Filter & Extract Relevant Events

**Overview:**  
Processes each garage sale event individually by extracting relevant details, cleaning data, and filtering based on proximity and event rank (importance).

**Nodes Involved:**  
- Loop Over Items (Split In Batches)  
- Get each Garage Sale info (HTML Extract)  
- Get Rank & Distance (Set)  
- Filter on close and bigger events (Filter)

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each event HTML block one by one to avoid large batch processing.  
  - Inputs: "Split Out"  
  - Outputs: Two outputs:  
    - Main output: "Filter on close and bigger events" (for filtered items)  
    - Secondary output: "Get each Garage Sale info" (for data extraction)  
  - Edge cases: Batch size defaults, possible delays if many items.

- **Get each Garage Sale info**  
  - Type: HTML Extract  
  - Role: Extracts from each event block:  
    - `City` (from `span.city`)  
    - `Distance` (from `span.dist`)  
    - `Rank` (from `span.dots`)  
  - Inputs: "Loop Over Items" (secondary output)  
  - Outputs: "Get Rank & Distance"  
  - Edge cases: Missing elements, HTML changes.

- **Get Rank & Distance**  
  - Type: Set node  
  - Role: Cleans and formats extracted data:  
    - Removes "km" suffix from Distance string and converts to number  
    - Replaces all "•" characters in Rank with "x" for uniformity  
  - Inputs: "Get each Garage Sale info"  
  - Outputs: "Loop Over Items" (main output) for further filtering  
  - Edge cases: Unexpected data formats causing parse errors.

- **Filter on close and bigger events**  
  - Type: Filter node  
  - Role: Filters events based on:  
    - Rank must be present (non-empty string)  
    - Distance ≤ 20 km  
  - Inputs: "Loop Over Items" (main output)  
  - Outputs: "Shape the response"  
  - Edge cases: Filter excludes all events (empty output).

---

#### 1.4 Format & Send Notifications via Telegram

**Overview:**  
Formats the filtered garage sale events into a readable message string and sends it via Telegram to the user.

**Nodes Involved:**  
- Shape the response (Set)  
- Set the message (Set)  
- Send an Alert (Telegram)

**Node Details:**

- **Shape the response**  
  - Type: Set node  
  - Role: Creates a formatted string field `Brocante` combining City, Rank, and Distance in the form `"City (Rank à Distance km)"`.  
  - Inputs: "Filter on close and bigger events"  
  - Outputs: "Set the message"  
  - Edge cases: Missing data fields; formatting issues.

- **Set the message**  
  - Type: Set node  
  - Role: Combines all formatted events into a single message string prefixed with an emoji and text:  
    ```
    📦🏡 Voici les brocantes : - {{ $json.Brocante }}
    ```  
  - Inputs: "Shape the response"  
  - Outputs: "Send an Alert"  
  - Edge cases: Empty input results in no message.

- **Send an Alert**  
  - Type: Telegram node  
  - Role: Sends the constructed message as a Telegram text message to a predefined chat ID.  
  - Key config:  
    - `chatId`: Set to user’s Telegram Chat ID ("Your_Chat_ID")  
    - `text`: Expression using `$json.message`  
    - Credential: Requires Telegram API credential configured in n8n  
  - Inputs: "Set the message"  
  - Outputs: None (end node)  
  - Edge cases: Telegram API failures, invalid chat ID, rate limits.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                               | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                          |
|-----------------------------|-----------------------|----------------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger        | Manual start trigger                          | None                             | Get location                   | ## 0️⃣ Daily Trigger & Geolocation Setup: Workflow trigger options and location fetching             |
| Every day at 7 AM           | Schedule Trigger      | Automatic daily time-based trigger            | None                             | Get location                   | ## 0️⃣ Daily Trigger & Geolocation Setup: Workflow trigger options and location fetching             |
| Get location                | Home Assistant        | Retrieves current user location               | Manual Trigger, Schedule Trigger | Set URL to parse               | ## 0️⃣ Daily Trigger & Geolocation Setup: Location fetched from Home Assistant                      |
| Set URL to parse            | Set                   | Builds dynamic URL for garage sales page      | Get location                    | Get Brocabrac                 | ## 0️⃣ Daily Trigger & Geolocation Setup: Constructs Brocabrac URL based on location                |
| Get Brocabrac               | HTTP Request          | Fetches HTML page from Brocabrac               | Set URL to parse                | Extract Date & Blocks          | ## 1️⃣ Fetch & Parse Garage Sale Data: Retrieves raw HTML from Brocabrac                            |
| Extract Date & Blocks       | HTML Extract          | Extracts event date and HTML event blocks      | Get Brocabrac                   | Any today?                    | ## 1️⃣ Fetch & Parse Garage Sale Data: Extracts date and block of garage sales                      |
| Any today?                  | If                    | Checks if events are scheduled for today       | Extract Date & Blocks           | Extract Garage Sales Events    | ## 1️⃣ Fetch & Parse Garage Sale Data: Filters events for current day                              |
| Extract Garage Sales Events | HTML Extract          | Extracts individual garage sales HTML elements | Any today? (true branch)        | Split Out                    | ## 1️⃣ Fetch & Parse Garage Sale Data: Extracts individual events                                  |
| Split Out                  | Split Out              | Splits events array into single event items    | Extract Garage Sales Events     | Loop Over Items               | ## 2️⃣ Process, Filter & Extract Relevant Events: Prepares events for individual processing        |
| Loop Over Items            | Split In Batches       | Processes each event block individually         | Split Out                      | Filter on close and bigger events, Get each Garage Sale info | ## 2️⃣ Process, Filter & Extract Relevant Events: Loops over each event                           |
| Get each Garage Sale info  | HTML Extract           | Extracts City, Distance, and Rank from event   | Loop Over Items (secondary)     | Get Rank & Distance           | ## 2️⃣ Process, Filter & Extract Relevant Events: Extracts detailed info per event                  |
| Get Rank & Distance        | Set                    | Cleans and formats Distance and Rank data      | Get each Garage Sale info       | Loop Over Items (main)        | ## 2️⃣ Process, Filter & Extract Relevant Events: Data cleanup and formatting                       |
| Filter on close and bigger events | Filter           | Filters events by distance ≤ 20 km and rank presence | Loop Over Items (main)          | Shape the response            | ## 2️⃣ Process, Filter & Extract Relevant Events: Applies proximity and rank filters                |
| Shape the response         | Set                    | Formats filtered events into summary strings   | Filter on close and bigger events | Set the message              | ## 3️⃣ Format & Send Notifications via Telegram: Prepares human-readable event descriptions        |
| Set the message            | Set                    | Combines event strings into final Telegram message | Shape the response             | Send an Alert                 | ## 3️⃣ Format & Send Notifications via Telegram: Message composition                               |
| Send an Alert              | Telegram                | Sends the formatted message to Telegram chat   | Set the message                | None                         | ## 3️⃣ Format & Send Notifications via Telegram: Sends alert to the user                           |
| Sticky Note                | Sticky Note             | Explanatory note                                | None                          | None                         | ## 1️⃣ Fetch & Parse Garage Sale Data: Details extraction and logic explanation                     |
| Sticky Note1               | Sticky Note             | Explanatory note                                | None                          | None                         | ## 2️⃣ Process, Filter & Extract Relevant Events: Data cleaning and filtering explanation          |
| Sticky Note2               | Sticky Note             | Explanatory note                                | None                          | None                         | ## 3️⃣ Format & Send Notifications via Telegram: Message formatting and sending explanation        |
| Sticky Note3               | Sticky Note             | Explanatory note                                | None                          | None                         | ## 0️⃣ Daily Trigger & Geolocation Setup: Trigger and geolocation details                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add **Manual Trigger** node named `"When clicking ‘Test workflow’"` for manual start.  
   - Add **Schedule Trigger** node named `"Every day at 7 AM"` with cron expression `"30 7 * * *"` for daily automatic runs.

2. **Set Up Location Fetching:**  
   - Add **Home Assistant** node named `"Get location"`. Configure with your Home Assistant credentials.  
   - Set `Entity ID` to your smartphone location sensor (e.g., `"Your_Smartphone_location_sensor"`).  
   - Use resource `"state"` to get current location data.

3. **Create URL Builder:**  
   - Add **Set** node named `"Set URL to parse"`.  
   - Add a string field `"URL"` with value expression:  
     ```
     https://brocabrac.fr/{{ $json.attributes.postal_code.slice(0,2) }}/{{ $json.attributes.locality }}
     ```  
   - Connect `"Get location"` output to this node.

4. **Fetch Garage Sales Data:**  
   - Add **HTTP Request** node named `"Get Brocabrac"`.  
   - Set URL as an expression: `{{$json.URL}}`.  
   - Set Response Format to `"String"`.  
   - Connect `"Set URL to parse"` output to this node.

5. **Extract Date and Garage Sales Blocks:**  
   - Add **HTML Extract** node named `"Extract Date & Blocks"`.  
   - Operation: `"extractHtmlContent"`.  
   - Extraction values:  
     - Key: `"Date"`, CSS selector: `div.block.ev-list > div:nth-child(1) > div.section-title`, return attribute `"data-date"`.  
     - Key: `"HTMLBlock"`, CSS selector: `div.block.ev-list > div`, return value `"html"`.  
   - Connect `"Get Brocabrac"` output to this node.

6. **Check for Today’s Events:**  
   - Add **If** node named `"Any today?"`.  
   - Condition: Check if `Date` equals today’s date (`$today.plus({days})`) using datetime equals operation.  
   - Connect `"Extract Date & Blocks"` output to this node.

7. **Extract Individual Garage Sale Events:**  
   - Add **HTML Extract** node named `"Extract Garage Sales Events"`.  
   - Data property: `"HTMLBlock"`.  
   - CSS selector: `div.ev`, return array of HTML.  
   - Connect `"Any today?"` true output to this node.

8. **Split Events for Processing:**  
   - Add **Split Out** node named `"Split Out"`.  
   - Field to split out: `"ev"`.  
   - Connect `"Extract Garage Sales Events"` output to this node.

9. **Loop Over Each Event:**  
   - Add **Split In Batches** node named `"Loop Over Items"`.  
   - Connect `"Split Out"` output to this node.

10. **Extract Details from Each Event:**  
    - Add **HTML Extract** node named `"Get each Garage Sale info"`.  
    - Data property: `"ev"`.  
    - Extract:  
      - `"City"` from `span.city`  
      - `"Distance"` from `span.dist`  
      - `"Rank"` from `span.dots`  
    - Connect secondary output of `"Loop Over Items"` to this node.

11. **Clean and Format Distance and Rank:**  
    - Add **Set** node named `"Get Rank & Distance"`.  
    - Assignments:  
      - `"Distance"` as number, expression: `{{$json.Distance.slice(0,-3)}}` (remove "km")  
      - `"Rank"` as string, expression: `{{$json.Rank.replaceAll('•','x')}}`  
    - Connect `"Get each Garage Sale info"` output to this node.

12. **Feed cleaned data back to loop:**  
    - Connect `"Get Rank & Distance"` output back to the main input of `"Loop Over Items"`.

13. **Filter Events by Distance and Rank:**  
    - Add **Filter** node named `"Filter on close and bigger events"`.  
    - Conditions:  
      - `"Rank"` contains (non-empty string)  
      - `"Distance"` ≤ 20  
    - Connect main output of `"Loop Over Items"` (after `"Get Rank & Distance"`) to this node.

14. **Format Filtered Events:**  
    - Add **Set** node named `"Shape the response"`.  
    - Assignment:  
      - `"Brocante"` string: `{{$json.City}} ({{$json.Rank}} à {{$json.Distance}} km)`  
    - Connect `"Filter on close and bigger events"` output to this node.

15. **Compose Final Message:**  
    - Add **Set** node named `"Set the message"`.  
    - Assignment:  
      - `"message"` string: `"📦🏡 Voici les brocantes : - {{$json.Brocante}}"`  
    - Connect `"Shape the response"` output to this node.

16. **Send Telegram Notification:**  
    - Add **Telegram** node named `"Send an Alert"`.  
    - Configure Telegram API credentials.  
    - Set `chatId` to your Telegram chat ID (replace `"Your_Chat_ID"`).  
    - Set message text as expression: `{{$json.message}}`.  
    - Connect `"Set the message"` output to this node.

17. **Connect triggers:**  
    - Connect both `"When clicking ‘Test workflow’"` and `"Every day at 7 AM"` nodes to `"Get location"`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow depends on a Home Assistant sensor providing up-to-date location data for accurate, location-aware garage sale queries. | Home Assistant integration and credentials are required.                                           |
| Telegram credentials must be configured in n8n with a valid bot token and chat ID for message delivery.                            | https://core.telegram.org/bots/api                                                                   |
| The Brocabrac.fr website’s HTML structure is critical for parsing; significant site changes may require workflow updates.           | Brocabrac.fr - Local garage sale listings                                                          |
| The workflow uses cron scheduling with timezone assumptions; verify server timezone matches user expectations for accurate alerts. | Cron expression: `"30 7 * * *"` triggering daily at 7:30 AM                                        |
| This workflow showcases advanced HTML parsing and dynamic filtering to transform static web data into actionable notifications.     | Workflow tagging: "Showcase"                                                                         |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.