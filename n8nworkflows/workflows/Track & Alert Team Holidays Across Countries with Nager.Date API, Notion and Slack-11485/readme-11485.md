Track & Alert Team Holidays Across Countries with Nager.Date API, Notion and Slack

https://n8nworkflows.xyz/workflows/track---alert-team-holidays-across-countries-with-nager-date-api--notion-and-slack-11485


# Track & Alert Team Holidays Across Countries with Nager.Date API, Notion and Slack

### 1. Workflow Overview

This workflow automates the tracking and alerting of team holidays across multiple countries using public holiday data from the Nager.Date API. It targets distributed teams working in different countries and helps prevent scheduling conflicts by notifying via Slack and syncing holiday data into a Notion database. The workflow is designed to run weekly (every Monday), fetch upcoming holidays within a configurable lookahead window, identify holidays shared between countries, then send alerts and update a central Notion calendar.

**Logical blocks:**

- **1.1 Schedule & Configuration**: Defines the weekly trigger, the list of countries to track, and the lookahead window in days.
- **1.2 Query Preparation**: Prepares the country-year combinations to query the public holiday API, covering the current and next year to handle year-end transitions.
- **1.3 Fetch & Filter Holidays**: Calls the Nager.Date API for each country-year pair, then filters holidays to keep only those within the lookahead window.
- **1.4 Find Shared Holidays & Format Data**: Aggregates holidays by date to identify shared holidays among countries and formats the data accordingly for Notion and Slack.
- **1.5 Notification & Sync**: Sends formatted holiday alerts to Slack and adds corresponding pages to a Notion database for team access.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Configuration

**Overview:**  
This block triggers the workflow weekly on Monday at 9 AM and sets up configuration parameters: the list of countries to track and the lookahead window in days.

**Nodes Involved:**  
- Weekly Schedule  
- Define Team Countries  
- Define Days to Lookahead

**Node Details:**

- **Weekly Schedule**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every Monday at 9:00 AM.  
  - Configuration: Interval set to weekly on day 1 (Monday) at 9 AM.  
  - Inputs: None (trigger node).  
  - Outputs: Connects to "Define Team Countries".  
  - Edge Cases: Trigger failure unlikely; depends on n8n scheduler service.  
  - Version: 1.1  

- **Define Team Countries**  
  - Type: Set  
  - Role: Defines an array of country codes (ISO 2-letter) to track holidays for.  
  - Configuration: Hardcoded array ['KR', 'MX', 'US'].  
  - Key Variables: `countries` array.  
  - Inputs: From "Weekly Schedule".  
  - Outputs: Connects to "Define Days to Lookahead".  
  - Edge Cases: Must ensure valid country codes conforming to Nager.Date API. Empty or wrong codes cause API errors.  
  - Version: 3.2  

- **Define Days to Lookahead**  
  - Type: Set  
  - Role: Defines how many days into the future to look for holidays.  
  - Configuration: Number field `Days` set to 50 by default.  
  - Inputs: From "Define Team Countries".  
  - Outputs: Connects to "Prepare Queries".  
  - Edge Cases: Setting this number too high might cause performance issues or API rate limiting.  
  - Version: 3.4  

---

#### 1.2 Query Preparation

**Overview:**  
Generates all combinations of countries and years (current and next year) to query the public holiday API, ensuring no holidays are missed during year transitions.

**Nodes Involved:**  
- Prepare Queries

**Node Details:**

- **Prepare Queries**  
  - Type: Code  
  - Role: Produces a list of objects each containing a country code and year (current and next year).  
  - Configuration: Uses JavaScript to loop over country codes and years.  
  - Key Expressions:  
    - `countries` from 'Define Team Countries' node.  
    - `currentYear` and `nextYear` calculated dynamically.  
  - Inputs: From "Define Days to Lookahead".  
  - Outputs: Connects to "Get Public Holidays".  
  - Edge Cases: Must ensure date retrieval logic is correct to avoid missing holidays.  
  - Version: 2  

---

#### 1.3 Fetch & Filter Holidays

**Overview:**  
Fetches public holidays for each country-year pair from the Nager.Date API and filters the results to keep only holidays within the configured lookahead window.

**Nodes Involved:**  
- Get Public Holidays  
- Filter Upcoming

**Node Details:**

- **Get Public Holidays**  
  - Type: HTTP Request  
  - Role: Calls Nager.Date API endpoint to retrieve public holidays for a given country and year.  
  - Configuration: URL dynamically constructed as `https://date.nager.at/api/v3/publicholidays/{{ $json.year }}/{{ $json.countryCode }}`.  
  - Inputs: From "Prepare Queries" (each item has `countryCode` and `year`).  
  - Outputs: Connects to "Filter Upcoming".  
  - Edge Cases: Possible failures include API downtime, rate limiting, invalid country codes, or malformed requests. Should handle HTTP errors gracefully.  
  - Version: 4.1  

- **Filter Upcoming**  
  - Type: Code  
  - Role: Filters the holiday list to keep only those falling between today and the future date defined by the lookahead window.  
  - Configuration: Reads `Days` from "Define Days to Lookahead".  
  - Key Expressions:  
    - Calculates `today` and `futureDate` (today + lookahead days).  
    - Filters holidays with dates in the inclusive range `[today, futureDate]`.  
  - Inputs: From "Get Public Holidays".  
  - Outputs: Connects to "Find Shared Holidays".  
  - Edge Cases: Timezone differences might affect date filtering; date format consistency is important.  
  - Version: 2  

---

#### 1.4 Find Shared Holidays & Format Data

**Overview:**  
Aggregates filtered holidays by date, identifies which holidays are shared among countries, and formats the data to include shared countries for Notion and Slack notifications.

**Nodes Involved:**  
- Find Shared Holidays

**Node Details:**

- **Find Shared Holidays**  
  - Type: Code  
  - Role:  
    1. Groups holidays by date.  
    2. Cleans date format to `YYYY-MM-DD` for Notion compatibility.  
    3. For each holiday, adds field `sharedWith` (comma-separated list of other countries sharing the same holiday date) for Notion.  
    4. Adds `sharedMsg` for Slack formatted message e.g., `(Also in: US, MX)`.  
  - Configuration:  
    - Processes all items from the "Filter Upcoming" node at once.  
    - Removes time portion from dates if present.  
  - Inputs: From "Filter Upcoming".  
  - Outputs: Connects to "Add to Notion".  
  - Edge Cases: Duplicate country entries for the same date are avoided. Empty shared lists handled gracefully (empty strings).  
  - Version: 2  

---

#### 1.5 Notification & Sync

**Overview:**  
Sends holiday alerts to Slack formatted with country, holiday name, date, and shared holiday info. Simultaneously adds or updates corresponding pages in a Notion database.

**Nodes Involved:**  
- Add to Notion  
- Notify Slack

**Node Details:**

- **Add to Notion**  
  - Type: Notion  
  - Role: Adds a new page to a Notion database recording the holiday with name, date, and shared countries fields.  
  - Configuration:  
    - Database ID is linked to a Notion database with properties:  
      - `Name` (Title) set as "`{{ holiday name }} ({{ countryCode }})`"  
      - `Date` (Date) set as holiday date  
      - `Shared Countries` (Rich Text) set as `sharedWith` string  
    - Credentials: Uses preconfigured Notion API credentials.  
  - Inputs: From "Find Shared Holidays".  
  - Outputs: Connects to "Notify Slack".  
  - Edge Cases: Requires valid Notion API credentials and properly configured database schema. API rate limits and write failures are possible.  
  - Version: 2.2  

- **Notify Slack**  
  - Type: Slack  
  - Role: Sends a message to a configured Slack channel alerting team members of upcoming holidays.  
  - Configuration:  
    - Message text template:  
      `🚢 *Team Alert:* \nNext {{ countryCode }} Holiday: *{{ name }}* is on {{ date }}.{{ sharedMsg }}`  
    - Channel selected via UI or variable binding.  
    - Uses a Slack webhook configured in credentials.  
  - Inputs: From "Add to Notion".  
  - Outputs: None (end of workflow).  
  - Edge Cases: Slack webhook must be valid with necessary permissions. Network issues or invalid channel IDs can cause failures.  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                                     | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                           |
|----------------------|--------------------|----------------------------------------------------|-------------------------|------------------------|-----------------------------------------------------------------------------------------------------|
| Weekly Schedule      | Schedule Trigger   | Triggers workflow weekly on Monday 9 AM            | None                    | Define Team Countries   | ## 1. Schedule & Configuration<br>This section triggers the workflow every Monday and sets variables |
| Define Team Countries | Set                | Defines list of countries to track                  | Weekly Schedule          | Define Days to Lookahead|                                                                                                     |
| Define Days to Lookahead | Set             | Defines lookahead window in days                     | Define Team Countries    | Prepare Queries         |                                                                                                     |
| Prepare Queries      | Code               | Generates country-year pairs for API queries        | Define Days to Lookahead | Get Public Holidays     | ## 2. Query Preparation<br>Prepares country+year combos for current and next year                    |
| Get Public Holidays  | HTTP Request       | Fetches public holidays from Nager.Date API         | Prepare Queries          | Filter Upcoming         | ## 3. Fetch & Filter<br>Queries API and filters holidays within lookahead window                    |
| Filter Upcoming      | Code               | Filters holidays to those within lookahead window   | Get Public Holidays      | Find Shared Holidays    |                                                                                                     |
| Find Shared Holidays | Code               | Aggregates holidays by date, finds shared holidays  | Filter Upcoming          | Add to Notion           | ## 3. Fetch & Filter<br>Aggregates results to identify shared holidays, formats date, updates Notion/Slack |
| Add to Notion        | Notion             | Adds/updates holiday entries in Notion database     | Find Shared Holidays     | Notify Slack            |                                                                                                     |
| Notify Slack         | Slack              | Sends alert messages to Slack channel                | Add to Notion            | None                   |                                                                                                     |
| Sticky Note3         | Sticky Note        | Explains Fetch & Filter block                        | None                    | None                   | ## 3. Fetch & Filter<br>Queries the API for holidays and filters results                            |
| Sticky Note1         | Sticky Note        | Explains Schedule & Configuration block             | None                    | None                   | ## 1. Schedule & Configuration<br>Triggers the workflow and sets config variables                   |
| Sticky Note2         | Sticky Note        | Explains Query Preparation block                     | None                    | None                   | ## 2. Query Preparation<br>Prepares country+year combos                                             |
| Sticky Note4         | Sticky Note        | Disabled - explains Compare & Sync block             | None                    | None                   | ## 3. Compare & Sync<br>Disabled: Aggregates and syncs results to Notion/Slack                      |
| Sticky Note          | Sticky Note        | Disabled - general workflow description and setup   | None                    | None                   | ## 🌍 Global Holiday Sync<br>Explains overall workflow, setup steps, and external links            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Weekly Schedule`  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday at 9:00 AM.  

3. **Add a Set node:**  
   - Name: `Define Team Countries`  
   - Define a field named `countries` as an array value: `['KR', 'MX', 'US']` (modify as needed).  
   - Connect `Weekly Schedule` → `Define Team Countries`.  

4. **Add another Set node:**  
   - Name: `Define Days to Lookahead`  
   - Define a numeric field `Days` and set it to `50` (or desired number of days).  
   - Connect `Define Team Countries` → `Define Days to Lookahead`.  

5. **Add a Code node:**  
   - Name: `Prepare Queries`  
   - Code:  
     ```javascript
     const countries = $('Define Team Countries').first().json.countries;
     const currentYear = new Date().getFullYear();
     const nextYear = currentYear + 1;
     const results = [];
     for (const code of countries) {
       results.push({ json: { countryCode: code, year: currentYear } });
       results.push({ json: { countryCode: code, year: nextYear } });
     }
     return results;
     ```  
   - Connect `Define Days to Lookahead` → `Prepare Queries`.  

6. **Add an HTTP Request node:**  
   - Name: `Get Public Holidays`  
   - URL: `https://date.nager.at/api/v3/publicholidays/{{ $json.year }}/{{ $json.countryCode }}` (use expression mode).  
   - Method: GET  
   - Connect `Prepare Queries` → `Get Public Holidays`.  

7. **Add a Code node:**  
   - Name: `Filter Upcoming`  
   - Code:  
     ```javascript
     const daysAhead = $('Define Days to Lookahead').first().json.Days;
     const items = $input.all();
     const today = new Date();
     const futureDate = new Date();
     futureDate.setDate(today.getDate() + daysAhead);
     return items.filter(item => {
       const holidayDate = new Date(item.json.date);
       return holidayDate >= today && holidayDate <= futureDate;
     });
     ```  
   - Connect `Get Public Holidays` → `Filter Upcoming`.  

8. **Add a Code node:**  
   - Name: `Find Shared Holidays`  
   - Code:  
     ```javascript
     const allItems = $('Filter Upcoming').all().map(i => i.json);
     const dateMap = {};
     for (const item of allItems) {
       let dateStr = item.date;
       if (dateStr.includes('T')) {
         dateStr = dateStr.split('T')[0];
       }
       item.date = dateStr;
       if (!dateMap[dateStr]) dateMap[dateStr] = [];
       if (!dateMap[dateStr].includes(item.countryCode)) {
         dateMap[dateStr].push(item.countryCode);
       }
     }
     const enrichedItems = allItems.map(item => {
       const dateStr = item.date;
       const shared = dateMap[dateStr].filter(c => c !== item.countryCode);
       item.sharedWith = shared.length ? shared.join(', ') : '';
       item.sharedMsg = shared.length ? ` (Also in: ${shared.join(', ')})` : '';
       return { json: item };
     });
     return enrichedItems;
     ```  
   - Connect `Filter Upcoming` → `Find Shared Holidays`.  

9. **Add a Notion node:**  
   - Name: `Add to Notion`  
   - Resource: Database Page  
   - Database ID: Select your Notion database (must have properties: Name (title), Date (date), Shared Countries (rich text))  
   - Properties:  
     - Name: `={{ $json.name }} ({{ $json.countryCode }})`  
     - Date: `={{ $json.date }}`  
     - Shared Countries: `={{ $json.sharedWith }}`  
   - Credentials: Map to your Notion API credentials  
   - Connect `Find Shared Holidays` → `Add to Notion`.  

10. **Add a Slack node:**  
    - Name: `Notify Slack`  
    - Operation: Send Message  
    - Channel: Set the Slack channel or use variable binding  
    - Text:  
      ```text
      🚢 *Team Alert:* 
      Next {{ $json.countryCode }} Holiday: *{{ $json.name }}* is on {{ $json.date }}.{{ $json.sharedMsg }}
      ```  
    - Credentials: Use your Slack webhook credentials  
    - Connect `Add to Notion` → `Notify Slack`.  

11. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                            | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow prevents scheduling conflicts by syncing distributed team holidays using the free, public Nager.Date API for holiday data.                                                                               | Workflow purpose                                                                                         |
| The Notion database must have properties: `Name` (Title), `Date` (Date), and `Shared Countries` (Rich Text) to properly store holiday records.                                                                         | Notion setup instructions                                                                                |
| Update 'Define Team Countries' node with appropriate ISO 2-letter country codes for your team regions.                                                                                                                  | Configuration step                                                                                       |
| The lookahead window in 'Define Days to Lookahead' node controls how far in the future holidays are considered; adjust according to team needs balancing API calls and alert relevance.                                  | Configuration guideline                                                                                  |
| Slack webhook credentials must be configured with permissions to post messages to the selected channel.                                                                                                                 | Slack integration setup                                                                                   |
| The workflow runs once weekly to minimize API usage and avoid spamming Slack and Notion.                                                                                                                                | Scheduling rationale                                                                                      |
| Disabled sticky notes contain useful explanations of workflow blocks for maintainers and can be enabled if needed for documentation purposes.                                                                          | Documentation notes                                                                                       |
| Nager.Date API documentation: https://date.nager.at/swagger/index.html                                                                                                                                                   | Official API reference                                                                                   |
| Notion API documentation: https://developers.notion.com/docs/getting-started                                                                                                                                             | Notion integration guide                                                                                  |
| Slack API and webhook info: https://api.slack.com/messaging/webhooks                                                                                                                                                     | Slack integration guide                                                                                   |

---

**Disclaimer:** The provided text is exclusively generated and analyzed from an n8n automation workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All handled data is lawful and public.