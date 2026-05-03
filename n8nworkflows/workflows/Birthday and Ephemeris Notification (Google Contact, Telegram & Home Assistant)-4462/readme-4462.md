Birthday and Ephemeris Notification (Google Contact, Telegram & Home Assistant)

https://n8nworkflows.xyz/workflows/birthday-and-ephemeris-notification--google-contact--telegram---home-assistant--4462


# Birthday and Ephemeris Notification (Google Contact, Telegram & Home Assistant)

### 1. Workflow Overview

This workflow automates the process of sending birthday and saint’s day notifications by integrating Google Contacts, the Nominis API (for saints of the day), Telegram messaging, and Home Assistant for voice announcements. It is designed for personal or family use cases where users want to be reminded of upcoming birthdays and relevant saint celebrations, with notifications delivered via Telegram and smart home speakers.

The workflow is logically divided into the following main blocks:

- **1.1 Scheduling & Date Preparation:** Trigger the workflow daily at 7:30 AM and calculate a date offset (3 days ahead) to look for upcoming celebrations.
- **1.2 Google Contacts Data Retrieval:** Fetch all contacts’ birthdays and first names from Google Contacts.
- **1.3 Saint of the Day Retrieval:** Query the Nominis API for saints corresponding to the offset date.
- **1.4 Data Aggregation & Matching:** Aggregate birthdays and first names, combine them with the saint names, and check if any first name matches a saint of the day.
- **1.5 Message Composition:** Create personalized messages for birthdays and saint celebrations.
- **1.6 Notification Delivery:** Send the composed messages via Telegram and announce them through a Google Home Speaker via Home Assistant.
- **1.7 Conditional Logic:** Handle scenarios such as no saint today or no birthdays, and ensure only relevant notifications are sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Date Preparation

- **Overview:** This block triggers the workflow daily at a fixed time and sets a date offset to check for upcoming birthdays and saints.
- **Nodes Involved:**  
  - `Everyday at 7am`  
  - `Set Date Offset`  
  - `Sticky Note`

- **Node Details:**

  - **Everyday at 7am**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression set to 7:30 AM daily (`30 7 * * *`)  
    - Inputs: None (trigger node)  
    - Outputs: Triggers `Set Date Offset`  
    - Edge Cases: Trigger failure (rare), cron misconfiguration  
    - Notes: Ensures automation runs every morning.

  - **Set Date Offset**  
    - Type: Set  
    - Configuration: Assigns a new field `date` as today’s date plus 3 days (`$today.plus({days:3})`)  
    - Inputs: Trigger from schedule  
    - Outputs: Feeds Google Contacts and Nominis URL nodes  
    - Edge Cases: Date calculation errors, timezone considerations  
    - Notes: Used to look ahead for upcoming events.

  - **Sticky Note**  
    - Content:  
      ```
      ## Scheduling & Offset
      If you need to put some time interval between now and the time of trigger
      ```
    - Purpose: Inform users about the scheduling logic and date offset use.

---

#### 2.2 Google Contacts Data Retrieval

- **Overview:** Retrieves all contacts from Google Contacts, focusing on birthdays and first names.
- **Nodes Involved:**  
  - `Google Contacts`  
  - `First names`  
  - `Birthday Today?`  
  - `Aggregate Birthdays`  
  - `Aggregate No Birthday`  
  - `Single list Birthday`  
  - `Get One first name list`  
  - `Sticky Note2`

- **Node Details:**

  - **Google Contacts**  
    - Type: Google Contacts node  
    - Configuration:  
      - Operation: Get All contacts  
      - Fields: birthdays, names  
      - Return All: true  
      - Raw Data: true (full data)  
    - Credentials: Google OAuth2 credentials  
    - Inputs: From `Set Date Offset`  
    - Outputs: Two branches: to `First names` and `Birthday Today?`  
    - Edge Cases: OAuth authentication errors, API rate limits, empty contact lists

  - **First names**  
    - Type: Aggregate  
    - Configuration: Aggregates `names[0].givenName` from contacts to extract first names  
    - Inputs: From `Google Contacts`  
    - Outputs: To `Get One first name list`  
    - Edge Cases: Empty or missing names field

  - **Get One first name list**  
    - Type: Set  
    - Configuration: Creates a unique list of first names by merging `name` and `givenName` fields and removing duplicates  
    - Inputs: From `First names`  
    - Outputs: To `Combine Firstname & Saints`  
    - Edge Cases: Missing or malformed name data

  - **Birthday Today?**  
    - Type: If  
    - Configuration: Checks if any contact’s birthday matches the offset date (month and day)  
    - Inputs: From `Google Contacts`  
    - Outputs: If true, to `Aggregate Birthdays`; if false, to `Aggregate No Birthday`  
    - Edge Cases: Contacts without birthdays, date mismatch due to timezone or format differences

  - **Aggregate Birthdays**  
    - Type: Aggregate  
    - Configuration: Aggregates `names[0].displayName` of contacts with birthday today  
    - Inputs: From `Birthday Today?` (true branch)  
    - Outputs: To `Single list Birthday`  
    - Edge Cases: No birthdays present

  - **Single list Birthday**  
    - Type: Set  
    - Configuration: Creates an array field `anniversary` merging `displayName` and `name` fields for birthday contacts  
    - Inputs: From `Aggregate Birthdays`  
    - Outputs: To `Birthday celebration message`  
    - Edge Cases: Incomplete name fields

  - **Aggregate No Birthday**  
    - Type: Aggregate  
    - Configuration: Aggregates `name` fields for contacts without birthdays today  
    - Inputs: From `Birthday Today?` (false branch)  
    - Outputs: To `Merge Birthday + Fête Messages`  
    - Edge Cases: No contacts or empty names

  - **Sticky Note2**  
    - Content:  
      ```
      ## Birthday & First name Operations
      Manipulation fo find first name & birthday dates and see if there are some matches
      ```
    - Purpose: Explains this block’s role in extracting and matching birthdays and first names.

---

#### 2.3 Saint of the Day Retrieval

- **Overview:** Fetches the list of saints for the offset date from the Nominis API.
- **Nodes Involved:**  
  - `Get Nominis URL`  
  - `Nominis - Saints du jour`  
  - `List of Saints of the day`  
  - `Sticky Note1`

- **Node Details:**

  - **Get Nominis URL**  
    - Type: Set  
    - Configuration: Builds the API URL based on the offset date’s day and month extracted via string slicing  
    - Inputs: From `Set Date Offset`  
    - Outputs: To `Nominis - Saints du jour`  
    - Edge Cases: Date string format variations

  - **Nominis - Saints du jour**  
    - Type: HTTP Request  
    - Configuration: GET request to the URL provided by `Get Nominis URL`  
    - Inputs: From `Get Nominis URL`  
    - Outputs: To `List of Saints of the day`  
    - Edge Cases: API downtime, network errors, malformed JSON responses

  - **List of Saints of the day**  
    - Type: Set  
    - Configuration: Extracts and merges major and derived saint names from the API response into a unique array field `Saints`  
    - Inputs: From `Nominis - Saints du jour`  
    - Outputs: To `Combine Firstname & Saints`  
    - Edge Cases: Empty or missing saint data

  - **Sticky Note1**  
    - Content:  
      ```
      ## Call Nominis API
      API to get information from the Saint of the day
      [More Info](https://nominis.cef.fr/contenus/integrations.html)
      ```
    - Purpose: Documentation and link to Nominis API integration details.

---

#### 2.4 Data Aggregation & Matching

- **Overview:** Combines first names from Google Contacts and saint names, then checks if any first name matches a saint of the day.
- **Nodes Involved:**  
  - `Combine Firstname & Saints`  
  - `Check if any firstname match a Saints of the day`  
  - `No Saint Today ?`

- **Node Details:**

  - **Combine Firstname & Saints**  
    - Type: Merge  
    - Configuration: Combines two input arrays (`prénoms` and `Saints`) into one merged dataset  
    - Inputs: From `Get One first name list` and `List of Saints of the day`  
    - Outputs: To `Check if any firstname match a Saints of the day`  
    - Edge Cases: Empty input on either side

  - **Check if any firstname match a Saints of the day**  
    - Type: Code  
    - Configuration:  
      - JavaScript code normalizes strings by removing accents and lowercasing, then compares first names with saint names.  
      - Outputs matches array and a boolean `hasMatch`  
    - Inputs: From `Combine Firstname & Saints`  
    - Outputs: To `No Saint Today ?`  
    - Edge Cases: String normalization errors, empty input arrays

  - **No Saint Today ?**  
    - Type: If  
    - Configuration: Checks if `hasMatch` is false (no matching saint names)  
    - Inputs: From `Check if any firstname match a Saints of the day`  
    - Outputs:  
      - True branch: To `Merge Birthday + Fête Messages` (skip saint message)  
      - False branch: To `"Bonne fête" celebration message` (compose saint celebration)  
    - Edge Cases: Boolean logic errors

---

#### 2.5 Message Composition

- **Overview:** Constructs personalized text messages for birthdays, saint celebrations, and combines them for final notification.
- **Nodes Involved:**  
  - `"Bonne fête" celebration message`  
  - `Birthday celebration message`  
  - `Merge Birthday + Fête Messages`  
  - `Check if any celebration to make`  
  - `Compose Message`  
  - `Sticky Note3`

- **Node Details:**

  - **"Bonne fête" celebration message**  
    - Type: Set  
    - Configuration: Creates a string field `fete` with a message listing saint matches, e.g., `Bonne fête X, Y`  
    - Inputs: From `No Saint Today ?` (false branch)  
    - Outputs: To `Merge Birthday + Fête Messages` (second input)  
    - Edge Cases: Empty matches array

  - **Birthday celebration message**  
    - Type: Set  
    - Configuration: Creates a string field `Birthday` with a message listing birthdays, e.g., `Aujourd'hui, c'est l'anniversaire de X, Y`  
    - Inputs: From `Single list Birthday`  
    - Outputs: To `Merge Birthday + Fête Messages` (first input)  
    - Edge Cases: Empty anniversary list

  - **Merge Birthday + Fête Messages**  
    - Type: Merge  
    - Configuration: Combines birthday and fête messages into one dataset  
    - Inputs: From `Birthday celebration message` and either `Aggregate No Birthday` or `"Bonne fête" celebration message`  
    - Outputs: To `Check if any celebration to make`  
    - Edge Cases: Merging empty messages

  - **Check if any celebration to make**  
    - Type: If  
    - Configuration: Checks if either `Birthday` or `fete` messages exist (non-empty)  
    - Inputs: From `Merge Birthday + Fête Messages`  
    - Outputs: If true, to `Compose Message`  
    - Edge Cases: False negatives if fields missing

  - **Compose Message**  
    - Type: Set  
    - Configuration: Combines both messages into a final `message` string with the offset date formatted in French locale, e.g.:  
      ```
      Nous sommes le 10 Oct 2025:
      [Birthday message]
      [Fête message]
      ```  
    - Inputs: From `Check if any celebration to make`  
    - Outputs: To `Sent Telegram Message` and `Send to Google Home Speaker`  
    - Edge Cases: Date formatting errors

  - **Sticky Note3**  
    - Content:  
      ```
      ## Celebration message composition
      ```
    - Purpose: Explains the purpose of this block.

---

#### 2.6 Notification Delivery

- **Overview:** Sends the composed message to a Telegram chat and uses Home Assistant to speak it aloud via a Google Home speaker.
- **Nodes Involved:**  
  - `Sent Telegram Message`  
  - `Send to Google Home Speaker`

- **Node Details:**

  - **Sent Telegram Message**  
    - Type: Telegram node  
    - Configuration:  
      - Sends text from `message` field  
      - Requires chat ID (to be set manually)  
      - OAuth credentials for Telegram API  
      - Append Attribution disabled  
    - Inputs: From `Compose Message`  
    - Outputs: None  
    - Edge Cases: Invalid chat ID, Telegram API rate limits, network errors

  - **Send to Google Home Speaker**  
    - Type: Home Assistant node  
    - Configuration:  
      - Domain: `tts`  
      - Service: `speak`  
      - Service attributes:  
        - `entity_id`: `tts.google_translate_fr_fr`  
        - `message`: message text  
        - `media_player_entity_id`: `media_player.google_cuisine` (Google Home speaker entity)  
      - Requires Home Assistant API credentials  
    - Inputs: From `Compose Message`  
    - Outputs: None  
    - Edge Cases: Home Assistant connectivity issues, invalid entity IDs, TTS service errors

---

### 3. Summary Table

| Node Name                           | Node Type                | Functional Role                              | Input Node(s)                        | Output Node(s)                                  | Sticky Note                                                                                          |
|-----------------------------------|--------------------------|----------------------------------------------|------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Everyday at 7am                   | Schedule Trigger         | Starts workflow daily at 7:30 AM              | None                               | Set Date Offset                                 | If you need to put some time interval between now and the time of trigger                          |
| Set Date Offset                   | Set                      | Sets the date offset to today + 3 days       | Everyday at 7am                   | Google Contacts, Get Nominis URL                 | If you need to put some time interval between now and the time of trigger                          |
| Google Contacts                  | Google Contacts          | Retrieves contacts with birthdays and names  | Set Date Offset                   | First names, Birthday Today?                     | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| First names                     | Aggregate                | Extracts first names from contacts            | Google Contacts                  | Get One first name list                          | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| Get One first name list          | Set                      | Creates unique first name list                 | First names                     | Combine Firstname & Saints                       | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| Birthday Today?                 | If                       | Checks if any birthday matches offset date    | Google Contacts                  | Aggregate Birthdays (true), Aggregate No Birthday (false) | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| Aggregate Birthdays             | Aggregate                | Aggregates display names of birthday contacts | Birthday Today? (true)           | Single list Birthday                            | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| Single list Birthday            | Set                      | Creates birthday anniversary list              | Aggregate Birthdays             | Birthday celebration message                    | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| Aggregate No Birthday           | Aggregate                | Aggregates names of contacts without birthdays | Birthday Today? (false)          | Merge Birthday + Fête Messages                   | ## Birthday & First name Operations Manipulation fo find first name & birthday dates and see if there are some matches |
| Get Nominis URL                 | Set                      | Constructs API URL for saint of the day        | Set Date Offset                 | Nominis - Saints du jour                        | ## Call Nominis API API to get information from the Saint of the day [More Info](https://nominis.cef.fr/contenus/integrations.html) |
| Nominis - Saints du jour        | HTTP Request             | Calls the Nominis API to get saints            | Get Nominis URL                | List of Saints of the day                       | ## Call Nominis API API to get information from the Saint of the day [More Info](https://nominis.cef.fr/contenus/integrations.html) |
| List of Saints of the day       | Set                      | Extracts and deduplicates saint names          | Nominis - Saints du jour        | Combine Firstname & Saints                      | ## Call Nominis API API to get information from the Saint of the day [More Info](https://nominis.cef.fr/contenus/integrations.html) |
| Combine Firstname & Saints      | Merge                    | Combines first names and saint names           | Get One first name list, List of Saints of the day | Check if any firstname match a Saints of the day |                                                                                                    |
| Check if any firstname match a Saints of the day | Code                     | Finds matches between first names and saints  | Combine Firstname & Saints       | No Saint Today ?                                |                                                                                                    |
| No Saint Today ?                | If                       | Checks if any saint matches were found         | Check if any firstname match a Saints of the day | "Bonne fête" celebration message (false), Merge Birthday + Fête Messages (true) |                                                                                                    |
| "Bonne fête" celebration message | Set                      | Creates saint celebration message               | No Saint Today ? (false)         | Merge Birthday + Fête Messages                   | ## Celebration message composition                                                                 |
| Birthday celebration message    | Set                      | Creates birthday celebration message            | Single list Birthday            | Merge Birthday + Fête Messages                   | ## Celebration message composition                                                                 |
| Merge Birthday + Fête Messages | Merge                    | Combines birthday and saint celebration messages | Birthday celebration message, Aggregate No Birthday or "Bonne fête" celebration message | Check if any celebration to make                  | ## Celebration message composition                                                                 |
| Check if any celebration to make | If                       | Checks if any message exists to send            | Merge Birthday + Fête Messages  | Compose Message                                 | ## Celebration message composition                                                                 |
| Compose Message                | Set                      | Combines final message with formatted date     | Check if any celebration to make | Sent Telegram Message, Send to Google Home Speaker | ## Celebration message composition                                                                 |
| Sent Telegram Message          | Telegram                 | Sends notification to Telegram chat             | Compose Message                | None                                            |                                                                                                    |
| Send to Google Home Speaker    | Home Assistant           | Sends TTS notification to Google Home speaker   | Compose Message                | None                                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Everyday at 7am`  
   - Type: Schedule Trigger  
   - Set Cron Expression: `30 7 * * *` (7:30 AM daily)

2. **Create a Set node**  
   - Name: `Set Date Offset`  
   - Add field: `date` (string)  
   - Value: `={{ $today.plus({days:3}) }}` (today’s date plus 3 days)  
   - Connect from `Everyday at 7am`

3. **Create a Google Contacts node**  
   - Name: `Google Contacts`  
   - Operation: `getAll`  
   - Fields: `birthdays`, `names`  
   - Return All: enabled  
   - Raw Data: enabled  
   - Set credentials for Google Contacts OAuth2  
   - Connect from `Set Date Offset`

4. **Create an Aggregate node**  
   - Name: `First names`  
   - Field to aggregate: `names[0].givenName`  
   - Connect from `Google Contacts`

5. **Create a Set node**  
   - Name: `Get One first name list`  
   - Add field: `prénoms` (string)  
   - Value:  
     ```
     ={{ Array.from(new Set([...( $json.name || [] ), ...( $json.givenName || [] )])) }}
     ```  
   - Connect from `First names`

6. **Create a Set node**  
   - Name: `Get Nominis URL`  
   - Add field: `URL` (string)  
   - Value:  
     ```
     =https://nominis.cef.fr/json/nominis.php?jour={{ $json.date.slice(8,10) }}&mois={{ $json.date.slice(5,7) }}
     ```  
   - Connect from `Set Date Offset`

7. **Create an HTTP Request node**  
   - Name: `Nominis - Saints du jour`  
   - HTTP Method: GET  
   - URL: Use `{{ $json.URL }}` from previous node  
   - Connect from `Get Nominis URL`

8. **Create a Set node**  
   - Name: `List of Saints of the day`  
   - Add field: `Saints` (array)  
   - Value:  
     ```
     ={{ Array.from(new Set([...Object.keys($json.response.prenoms.majeurs), ...Object.keys($json.response.prenoms.derives)])) }}
     ```  
   - Connect from `Nominis - Saints du jour`

9. **Create a Merge node**  
   - Name: `Combine Firstname & Saints`  
   - Mode: Combine All  
   - Connect inputs from `Get One first name list` and `List of Saints of the day`

10. **Create a Code node**  
    - Name: `Check if any firstname match a Saints of the day`  
    - JavaScript code:  
      ```js
      function normalizeString(str) {
        return str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase();
      }
      const saints = $json["Saints"].split(',').map(s => normalizeString(s.trim())).filter(s => s.length > 0);
      const givenNames = $json["prénoms"].split(',').map(s => normalizeString(s.trim())).filter(s => s.length > 0);
      const matches = saints.filter(name => givenNames.includes(name));
      return [{ json: { matches, hasMatch: matches.length > 0 } }];
      ```
    - Connect from `Combine Firstname & Saints`

11. **Create an If node**  
    - Name: `No Saint Today ?`  
    - Condition: Check if `hasMatch` is false  
    - Connect from `Check if any firstname match a Saints of the day`

12. **Create an If node**  
    - Name: `Birthday Today?`  
    - Conditions:  
      - `$json.birthdays[0].date.month == offset date month`  
      - `$json.birthdays[0].date.day == offset date day`  
    - Connect from `Google Contacts`

13. **Create two Aggregate nodes**  
    - Name: `Aggregate Birthdays` (for true branch of `Birthday Today?`)  
      - Aggregate `names[0].displayName`  
    - Name: `Aggregate No Birthday` (for false branch of `Birthday Today?`)  
      - Aggregate `name`

14. **Create a Set node**  
    - Name: `Single list Birthday`  
    - Add field: `anniversary` (array)  
    - Value:  
      ```
      = [].concat($json.displayName || [], $json.name || [])
      ```  
    - Connect from `Aggregate Birthdays`

15. **Create two Set nodes for messages**  
    - Name: `Birthday celebration message`  
      - Field: `Birthday` (string)  
      - Value:  
        ```
        =Aujourd'hui, c'est l'anniversaire de {{ $json.anniversary.join(', ') }}
        ```  
      - Connect from `Single list Birthday`  
    - Name: `"Bonne fête" celebration message`  
      - Field: `fete` (string)  
      - Value:  
        ```
        =Bonne fête {{ $json.matches.join(', ') }}
        ```  
      - Connect from `No Saint Today ?` (false branch)

16. **Create a Merge node**  
    - Name: `Merge Birthday + Fête Messages`  
    - Mode: Combine All  
    - Connect inputs from `Birthday celebration message` and `"Bonne fête" celebration message` (or `Aggregate No Birthday` if no saint)

17. **Create an If node**  
    - Name: `Check if any celebration to make`  
    - Condition: Check if either `Birthday` or `fete` fields exist and are not empty  
    - Connect from `Merge Birthday + Fête Messages`

18. **Create a Set node**  
    - Name: `Compose Message`  
    - Field: `message` (string)  
    - Value:  
      ```
      =Nous sommes le {{ $('Set Date Offset').item.json.date.toDateTime().setLocale('fr').format('dd LLL yyyy') }}:\n{{ $json.Birthday }}{{ $json.fete }}
      ```  
    - Connect from `Check if any celebration to make`

19. **Create a Telegram node**  
    - Name: `Sent Telegram Message`  
    - Text: `{{ $json.message }}`  
    - Chat ID: Enter your Telegram chat ID  
    - Credentials: Telegram API OAuth2  
    - Connect from `Compose Message`

20. **Create a Home Assistant node**  
    - Name: `Send to Google Home Speaker`  
    - Domain: `tts`  
    - Service: `speak`  
    - Service Attributes:  
      - `entity_id`: `tts.google_translate_fr_fr`  
      - `message`: `{{ $json.message }}`  
      - `media_player_entity_id`: `media_player.google_cuisine`  
    - Credentials: Home Assistant API  
    - Connect from `Compose Message`

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                         |
|------------------------------------------------------------------------------|------------------------------------------------------------------------|
| API to get information from the Saint of the day                            | [Nominis API Integration](https://nominis.cef.fr/contenus/integrations.html) |
| Scheduling & Offset: If you need to put some time interval between trigger and event | Scheduling block explanation                                           |
| Birthday & First name Operations: Manipulation to find first names and birthdays and check matches | Birthday and name processing block                                    |
| Celebration message composition                                              | Message building block                                                 |

---

**Disclaimer:** This workflow is designed exclusively for lawful and public data processing using n8n automation. It respects all content policies and contains no illegal, offensive, or protected information.