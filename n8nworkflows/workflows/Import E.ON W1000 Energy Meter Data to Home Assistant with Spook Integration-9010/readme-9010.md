Import E.ON W1000 Energy Meter Data to Home Assistant with Spook Integration

https://n8nworkflows.xyz/workflows/import-e-on-w1000-energy-meter-data-to-home-assistant-with-spook-integration-9010


# Import E.ON W1000 Energy Meter Data to Home Assistant with Spook Integration

### 1. Workflow Overview

This workflow automates the import of energy meter data exported by the E.ON W1000 device into Home Assistant, integrating via the Spook add-on to maintain long-term energy statistics and update meter entities. It is designed to process emailed Excel attachments from E.ON, parse and reshape the data, aggregate 15-minute increments into hourly totals, and update Home Assistant's recorder and input_number entities accordingly.

Logical blocks:

- **1.1 Triggers & Email Retrieval**: Handles incoming triggers either by Gmail webhook, IMAP polling, or scheduled polling to fetch relevant emails from E.ON with the energy meter data export.

- **1.2 Email & Attachment Validation**: Validates email subject and attachment presence/type to ensure correct data is processed.

- **1.3 Data Extraction & Normalization**: Reads Excel attachments, extracts and splits data columns, renames keys to normalized names, and merges multiple data sets by timestamp.

- **1.4 Date-Time Conversion & Hourly Aggregation**: Converts Excel serial dates to standard ISO datetime format, rounds to the nearest hour, and aggregates 15-minute data into hourly sums with cumulative meter calculations.

- **1.5 Payload Generation for Home Assistant**: Creates data lists formatted for Home Assistant’s `recorder.import_statistics` service for two meter channels.

- **1.6 Home Assistant Updates**: Calls the Spook integration to import statistics, and updates the last known meter values in Home Assistant input_number entities.

- **1.7 Auxiliary Nodes and Notes**: Includes informational sticky notes explaining the workflow logic, configuration tips, and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers & Email Retrieval

**Overview:** Listens for new E.ON emails containing W1000 exports or periodically polls recent messages to ensure data ingestion.

**Nodes involved:**

- `Gmail Trigger`
- `Email Trigger (IMAP)`
- `Schedule Trigger`
- `Get last 5 messages`
- `Aggregate_id`
- `Get a message[0]`

**Node details:**

- **Gmail Trigger**  
  - Type: Trigger node for Gmail  
  - Configured to filter emails from `noreply@eon.com`  
  - Fires on new matching emails every minute  
  - Credential: Gmail OAuth2  
  - Outputs JSON with email data  

- **Email Trigger (IMAP)**  
  - Type: IMAP email read trigger  
  - Configured with custom IMAP search for unseen emails from `noreply@eon.com` or subject containing `[EON-W1000]`  
  - Downloads attachments  
  - Credential: IMAP credentials  
  - Alternative to Gmail Trigger, useful if Gmail API unavailable  

- **Schedule Trigger**  
  - Type: Time-based trigger  
  - Fires daily at 14:00 (2 PM)  
  - Initiates batch fetch of last 5 messages from Gmail  

- **Get last 5 messages**  
  - Gmail node to retrieve last 5 emails from `noreply@eon.com`  
  - Used by Schedule Trigger to catch missed emails  
  - Credential: Gmail OAuth2  

- **Aggregate_id**  
  - Aggregate node to collect message IDs and internal dates into arrays for batch processing  
  - Input: output from Check Email Subject node  
  - Outputs aggregated IDs for next node to fetch individual messages  

- **Get a message[0]**  
  - Gmail node to fetch a single message by ID (first in aggregated list)  
  - Downloads attachments  
  - Credential: Gmail OAuth2  

**Edge cases / failure types:**

- Authentication failures on Gmail or IMAP nodes due to expired or revoked credentials  
- Network timeouts or API rate limits  
- No emails matching filters (safe, just no output)  
- Partial email data missing attachments  

---

#### 1.2 Email & Attachment Validation

**Overview:** Ensures emails have correct subject and includes an Excel attachment before processing.

**Nodes involved:**

- `Check Email Subject`
- `If attachment_0 is xlsx`
- `No Operation, do nothing1`

**Node details:**

- **Check Email Subject**  
  - If node checking if subject equals `[EON-W1000]` (case sensitive)  
  - Input varies: `$json.Subject` or `$json.subject` depending on node source  
  - On true: proceeds to aggregate message IDs  
  - On false: routes to No Operation  

- **If attachment_0 is xlsx**  
  - If node verifying:  
    - Attachment named `attachment_0` exists  
    - File extension is `xls` or `xlsx` (case insensitive)  
  - On true: proceeds to Extract from File node  
  - On false: routes to No Operation  

- **No Operation, do nothing1**  
  - NoOp node to safely terminate branches where conditions are not met  

**Edge cases / failure types:**

- Email without attachment or with unexpected file type triggers no processing  
- Subject mismatch leads to silent no-op (safe fail)  
- Case sensitivity in subject or extension could cause false negatives  

---

#### 1.3 Data Extraction & Normalization

**Overview:** Extracts Excel data, splits into multiple logical data sets, renames keys for uniformity, and merges data sets on timestamp.

**Nodes involved:**

- `Extract from File`
- `Extract default data from source (+A)`
- `Extract '*_1' data from source (-A)`
- `Extract '*_2' data from source (1_8_0)`
- `Extract '*_3' data from source (2_8_0)`
- `Rename "*_1" keys for merge`
- `Rename "*_1" keys for merge1`
- `Rename "*_1" keys for merge2`
- `Rename "*_1" keys for merge3`
- `Merge (+A; -A)`
- `Merge (+A; -A)1`
- `Merge (+A; -A)2`

**Node details:**

- **Extract from File**  
  - Extracts `.xlsx` file from binary property `attachment_0`  
  - Outputs JSON array for each sheet row  

- **Extract default data from source (+A)**  
  - SplitOut node extracts columns: `Időbélyeg` (timestamp), `Érték` (value) representing +A data  
  - Output: rows with these fields  

- **Extract '*_1' data from source (-A)**  
  - SplitOut node extracts `Időbélyeg`, `Érték_1` columns representing -A data  

- **Extract '*_2' data from source (1_8_0)**  
  - SplitOut node extracts `Időbélyeg`, `Érték_2` columns representing meter reading 1_8_0  

- **Extract '*_3' data from source (2_8_0)**  
  - SplitOut node extracts `Időbélyeg`, `Érték_3` columns representing meter reading 2_8_0  

- **Rename "*_1" keys for merge*** nodes (4 total)  
  - Rename keys to uniform names:  
    - `Időbélyeg` → `start` (timestamp)  
    - `Érték` or `Érték_1`, `Érték_2`, `Érték_3` → `AP`, `AM`, `1_8_0`, `2_8_0` respectively  
  - Helps merging by consistent field names  

- **Merge (+A; -A)**  
  - Combines +A and -A datasets on `start` timestamp using "combine" mode with joinMode "keepEverything"  
  - Subsequent merges include the 1_8_0 and 2_8_0 data similarly to form one unified dataset per timestamp  

**Edge cases / failure types:**

- Missing or malformed Excel columns cause empty or incorrect splits  
- Data type mismatches during renaming or merging  
- Mismatched timestamps leading to incomplete merges  
- Duplicate timestamps may cause data duplication or overwrite depending on merge config  

---

#### 1.4 Date-Time Conversion & Hourly Aggregation

**Overview:** Converts Excel serial date to standard datetime, normalizes timestamps to hourly granularity, and aggregates 15-minute increments into hourly sums with cumulative meter baseline corrections.

**Nodes involved:**

- `Convert Excel time`
- `Convert datetime to Spook format`
- `Calculate hourly sum and`

**Node details:**

- **Convert Excel time**  
  - DateTime node adding Excel serial day count (field `start`) to base date `1899-12-30`  
  - Converts Excel numeric date to ISO date string  
  - Keeps input fields included  

- **Convert datetime to Spook format**  
  - Formats `start` datetime to `yyyy-MM-dd HH:00:00ZZ` (zeroes minutes/seconds, includes timezone offset)  
  - Rounds down to top of hour for grouping  
  - Includes input fields  

- **Calculate hourly sum and**  
  - Code node processing all merged data items  
  - Groups items by hourly timestamp  
  - Sums AP (import +A) and AM (export -A) values per hour  
  - Tracks and resets baseline meter values (`1_8_0`, `2_8_0`) when present  
  - Outputs array of hourly data with cumulative meter readings and summed values  
  - Implements safe conversion of values to numbers and defaulting  
  - Sorts by time ascending  

**Edge cases / failure types:**

- Excel date conversion errors due to invalid serial numbers  
- Timezone mismatches between n8n and Home Assistant causing incorrect hour grouping  
- Missing meter baseline readings on first hour (initialized to zero)  
- Code errors if input data missing expected fields  

---

#### 1.5 Payload Generation for Home Assistant

**Overview:** Creates data payloads for Home Assistant's recorder import and aggregates statistics per meter.

**Nodes involved:**

- `Generate 1_8_0 list for stats`
- `Generate 2_8_0 list for stats`
- `Generate 1_8_0 stats`
- `Generate 2_8_0 stats`

**Node details:**

- **Generate 1_8_0 list for stats**  
  - Set node creating JSON objects per item with keys:  
    - `start`: JavaScript Date object from timestamp  
    - `state`: meter reading value from `1_8_0` field  
    - `sum`: same as state (required by recorder for total_increasing)  

- **Generate 2_8_0 list for stats**  
  - Similar to above but for `2_8_0` meter readings  

- **Generate 1_8_0 stats**  
  - Aggregate node aggregating all generated 1_8_0 list items into a single data array field for Home Assistant call  

- **Generate 2_8_0 stats**  
  - Similar aggregation for 2_8_0 list  

**Edge cases / failure types:**

- Invalid or missing meter state values leading to empty or corrupted statistics arrays  
- Date conversion errors affecting the `start` field  
- Aggregation output format must match Home Assistant expectations  

---

#### 1.6 Home Assistant Updates

**Overview:** Sends the processed statistics to Home Assistant via Spook integration and updates input_number entities with last known meter states.

**Nodes involved:**

- `Spook: update +A hitorical data1`
- `Spook: update -A hitorical data1`
- `Update input_number.import entity state1`
- `Update input_number.exportt entity state1`

**Node details:**

- **Spook: update +A hitorical data1**  
  - Home Assistant node calling `recorder.import_statistics` service in domain `recorder`  
  - Parameters:  
    - `statistic_id`: `sensor.grid_energy_import`  
    - `source`: `recorder`  
    - `unit_of_measurement`: `kWh`  
    - `has_mean`: false  
    - `has_sum`: true  
    - `stats`: aggregated data array from previous step  
  - Credential: Home Assistant API with long-lived token  

- **Spook: update -A hitorical data1**  
  - Similar call but for `sensor.grid_energy_export`  

- **Update input_number.import entity state1**  
  - Home Assistant node to upsert state of `input_number.grid_import_meter` entity  
  - State set to last state's meter value from 1_8_0 stats  

- **Update input_number.exportt entity state1**  
  - Upserts state of `input_number.grid_export_meter` entity  
  - State from last value of 2_8_0 stats  

**Edge cases / failure types:**

- Authentication failure with Home Assistant API (invalid token)  
- Home Assistant service call failures (e.g. recorder integration disabled)  
- Entity IDs mismatch or missing in HA configuration  
- Payload format errors causing import failure  

---

#### 1.7 Auxiliary Nodes and Notes

**Overview:** Sticky notes provide detailed explanations, configuration tips, troubleshooting advice, and best practices.

**Key content summary:**

- Subject and attachment validation notes  
- Column mapping and key renaming rationale  
- Time conversion details and timezone considerations  
- Hourly grouping and code node logic explanation  
- Home Assistant recorder import payload structure  
- Entity IDs and template sensor configuration guidance  
- Credentials setup recommendations with security emphasis  
- Troubleshooting common issues such as missing rows, timezone errors, HA import failures  
- Home Assistant prerequisites including Recorder and Spook integration installation  
- Example helper configurations for input_number and template sensors  

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                   | Input Node(s)                 | Output Node(s)                         | Sticky Note                                                                                                                                                           |
|----------------------------------|---------------------------|---------------------------------|------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                    | Gmail Trigger             | Incoming Gmail email trigger     |                              | Check Email Subject                   | ## Gmail trigger                                                                                                                                                     |
| Email Trigger (IMAP)             | EmailReadImap             | Incoming IMAP email trigger      |                              | If attachment_0 is xlsx               | ## IMAP trigger                                                                                                                                                      |
| Schedule Trigger                | Schedule Trigger          | Scheduled polling trigger daily  |                              | Get last 5 messages                  | ## Schedule trigger                                                                                                                                                   |
| Get last 5 messages             | Gmail                     | Batch fetch recent emails        | Schedule Trigger             | Check Email Subject                   |                                                                                                                                                                     |
| Aggregate_id                    | Aggregate                 | Aggregate message IDs            | Check Email Subject          | Get a message[0]                     |                                                                                                                                                                     |
| Get a message[0]                | Gmail                     | Fetch single message and attachments | Aggregate_id                | If attachment_0 is xlsx               |                                                                                                                                                                     |
| Check Email Subject             | If                        | Validate email subject           | Gmail Trigger, Get last 5 messages | Aggregate_id, No Operation          | # Subject & Attachment checks  - Gmail payload uses `Subject` (capital S), IMAP node uses `subject` (lowercase).  - This template handles both. See sticky note details. |
| If attachment_0 is xlsx         | If                        | Check attachment presence & type | Get a message[0], Email Trigger (IMAP) | Extract from File, No Operation       | See sticky note above                                                                                                                                                 |
| No Operation, do nothing1       | NoOp                      | Terminates invalid branches     | Check Email Subject, If attachment_0 is xlsx |                                    |                                                                                                                                                                     |
| Extract from File               | ExtractFromFile           | Parse Excel attachment contents | If attachment_0 is xlsx      | Extract default data..., Extract '*_1' data..., Extract '*_2' data..., Extract '*_3' data... |                                                                                                                                                                     |
| Extract default data from source (+A) | SplitOut                  | Extract +A columns              | Extract from File            | Rename "*_1" keys for merge3          | # Column Mapping See sticky note for detailed mapping                                                                                                               |
| Extract '*_1' data from source (-A) | SplitOut                  | Extract -A columns              | Extract from File            | Rename "*_1" keys for merge           | See sticky note above                                                                                                                                                 |
| Extract '*_2' data from source (1_8_0) | SplitOut                  | Extract meter 1_8_0 data       | Extract from File            | Rename "*_1" keys for merge1          | See sticky note above                                                                                                                                                 |
| Extract '*_3' data from source (2_8_0) | SplitOut                  | Extract meter 2_8_0 data       | Extract from File            | Rename "*_1" keys for merge2          | See sticky note above                                                                                                                                                 |
| Rename "*_1" keys for merge     | RenameKeys                | Normalize keys for merge (-A)    | Extract '*_1' data           | Merge (+A; -A)                       | See sticky note above                                                                                                                                                 |
| Rename "*_1" keys for merge1    | RenameKeys                | Normalize keys for merge (1_8_0) | Extract '*_2' data           | Merge (+A; -A)1                      | See sticky note above                                                                                                                                                 |
| Rename "*_1" keys for merge2    | RenameKeys                | Normalize keys for merge (2_8_0) | Extract '*_3' data           | Merge (+A; -A)1                      | See sticky note above                                                                                                                                                 |
| Rename "*_1" keys for merge3    | RenameKeys                | Normalize keys for merge (+A)     | Extract default data         | Merge (+A; -A)                       | See sticky note above                                                                                                                                                 |
| Merge (+A; -A)                 | Merge                    | Merge +A and -A datasets         | Rename "*_1" keys for merge, Rename "*_1" keys for merge3 | Merge (+A; -A)2                       | See sticky note above                                                                                                                                                 |
| Merge (+A; -A)1                | Merge                    | Merge 1_8_0 and 2_8_0 datasets  | Rename "*_1" keys for merge1, Rename "*_1" keys for merge2 | Merge (+A; -A)2                       | See sticky note above                                                                                                                                                 |
| Merge (+A; -A)2                | Merge                    | Merge all datasets into one      | Merge (+A; -A), Merge (+A; -A)1 | Convert Excel time                   | See sticky note above                                                                                                                                                 |
| Convert Excel time             | DateTime                  | Convert Excel serial date to ISO | Merge (+A; -A)2             | Convert datetime to Spook format     | # Time Conversion See sticky note for details                                                                                                                       |
| Convert datetime to Spook format | DateTime                  | Format date to Spook compatible  | Convert Excel time           | Calculate hourly sum and             | See sticky note above                                                                                                                                                 |
| Calculate hourly sum and        | Code                      | Aggregate 15-min data to hourly  | Convert datetime to Spook format | Generate 1_8_0 list for stats, Generate 2_8_0 list for stats | # Hourly Grouping Logic See sticky note for code explanation                                                                                                        |
| Generate 1_8_0 list for stats   | Set                       | Create 1_8_0 payload items for HA | Calculate hourly sum and     | Generate 1_8_0 stats                 | # recorder.import_statistics payload See sticky note                                                                                                                |
| Generate 2_8_0 list for stats   | Set                       | Create 2_8_0 payload items for HA | Calculate hourly sum and     | Generate 2_8_0 stats                 | See sticky note above                                                                                                                                                 |
| Generate 1_8_0 stats            | Aggregate                 | Aggregate 1_8_0 payload array    | Generate 1_8_0 list for stats | Spook: update +A hitorical data1     |                                                                                                                                                                     |
| Generate 2_8_0 stats            | Aggregate                 | Aggregate 2_8_0 payload array    | Generate 2_8_0 list for stats | Spook: update -A hitorical data1     |                                                                                                                                                                     |
| Spook: update +A hitorical data1 | HomeAssistant             | Import +A stats to Home Assistant | Generate 1_8_0 stats        | Update input_number.import entity state1 | # Entity IDs updated See sticky note                                                                                                                                |
| Spook: update -A hitorical data1 | HomeAssistant             | Import -A stats to Home Assistant | Generate 2_8_0 stats        | Update input_number.exportt entity state1 | See sticky note above                                                                                                                                                 |
| Update input_number.import entity state1 | HomeAssistant             | Update import meter entity state | Spook: update +A hitorical data1 |                                      | See sticky note above                                                                                                                                                 |
| Update input_number.exportt entity state1 | HomeAssistant             | Update export meter entity state | Spook: update -A hitorical data1 |                                      | See sticky note above                                                                                                                                                 |
| Sticky Note*                    | StickyNote                | Explanations, tips, troubleshooting | Various                    |                                    | Multiple sticky notes cover all workflow aspects, please refer to relevant notes for detailed context.                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers**

   - Add **Gmail Trigger** node:  
     - Credential: Gmail OAuth2 with read-only email access  
     - Filter: sender `noreply@eon.com`  
     - Poll interval: every minute  
   
   - Add **Email Trigger (IMAP)** node (optional):  
     - Credential: IMAP account with read-only access  
     - Custom email search: unseen emails from `noreply@eon.com` or subject containing `[EON-W1000]`  
     - Enable attachment download  

   - Add **Schedule Trigger** node:  
     - Set to run daily at 14:00 (2 PM)  
   
2. **Batch Fetch Last Messages (Schedule Trigger path)**

   - Add **Get last 5 messages** (Gmail node):  
     - Credential: Gmail OAuth2  
     - Filter: sender `noreply@eon.com`  
     - Limit: 5 messages  
   
3. **Validate Email Subject**

   - Add **Check Email Subject** (If node):  
     - Condition: `$json.Subject` or `$json.subject` equals `[EON-W1000]` (case sensitive)  
     - True: continue processing  
     - False: terminate with NoOp  

4. **Aggregate Message IDs (for batch processing)**

   - Add **Aggregate_id** (Aggregate node):  
     - Aggregate fields: `id` and `internalDate` into arrays  
     - Input: output of Check Email Subject true branch  

5. **Fetch Single Message**

   - Add **Get a message[0]** (Gmail node):  
     - Credential: Gmail OAuth2  
     - Message ID: expression `{{$json.id[0]}}` (first ID from aggregation)  
     - Download attachments set to true  

6. **Validate Attachment**

   - Add **If attachment_0 is xlsx** (If node):  
     - Conditions:  
       - Email subject equals `[EON-W1000]`  
       - Binary property `attachment_0` exists  
       - File extension is `.xls` or `.xlsx` (case insensitive)  
     - True: proceed  
     - False: terminate with NoOp  

7. **Extract Excel Data**

   - Add **Extract from File** (ExtractFromFile node):  
     - Operation: `xlsx`  
     - Binary Property: `attachment_0`  

8. **Split Out Data Columns**

   - Add four **SplitOut** nodes:  
     - Extract default (+A) data: columns `Időbélyeg,Érték`  
     - Extract '*_1' (-A) data: columns `Időbélyeg,Érték_1`  
     - Extract '*_2' (1_8_0) data: columns `Időbélyeg,Érték_2`  
     - Extract '*_3' (2_8_0) data: columns `Időbélyeg,Érték_3`  

9. **Rename Keys for Normalization**

   - Add four **RenameKeys** nodes to map columns to:  
     - `Időbélyeg` → `start` in all  
     - `Érték` → `AP` (default +A)  
     - `Érték_1` → `AM` (-A)  
     - `Érték_2` → `1_8_0` (meter 1)  
     - `Érték_3` → `2_8_0` (meter 2)  

10. **Merge Datasets**

    - Merge +A and -A datasets on `start` (field to match) in "combine" mode  
    - Merge meter 1 and meter 2 datasets similarly  
    - Merge the two merged outputs together into one unified dataset  

11. **Convert Excel Time to ISO**

    - Add **DateTime** node "Convert Excel time":  
      - Operation: add to date  
      - Add `start` field as days to base date `1899-12-30`  
      - Output field: `start`  

12. **Format Date for Spook**

    - Add **DateTime** node "Convert datetime to Spook format":  
      - Format date: `yyyy-MM-dd HH:00:00ZZ`  
      - Round to top of hour  
      - Output field: `start`  

13. **Aggregate Hourly Sums**

    - Add **Code** node "Calculate hourly sum and":  
      - Copy and paste provided JavaScript code  
      - Input: items with fields `{ start, AP, AM, 1_8_0?, 2_8_0? }`  
      - Output: grouped and cumulative hourly data  

14. **Generate Payload Lists for Home Assistant**

    - Add two **Set** nodes:  
      - For `1_8_0` list: create JSON with `start` (Date), `state`, `sum` from field `1_8_0`  
      - For `2_8_0` list: similar for field `2_8_0`  

15. **Aggregate Payloads**

    - Add two **Aggregate** nodes:  
      - Aggregate all items from the above lists into `data` field arrays  

16. **Send Data to Home Assistant (Spook Integration)**

    - Add two **Home Assistant** nodes:  
      - Domain: `recorder`  
      - Service: `import_statistics`  
      - Parameters:  
        - `statistic_id`: `sensor.grid_energy_import` for import, `sensor.grid_energy_export` for export  
        - `source`: `recorder`  
        - `unit_of_measurement`: `kWh`  
        - `has_mean`: false  
        - `has_sum`: true  
        - `stats`: aggregated data array from previous step  
      - Credential: Home Assistant API (Long-Lived Access Token)  

17. **Update Meter State Entities**

    - Add two **Home Assistant** nodes:  
      - Update `input_number.grid_import_meter` with last `1_8_0` state value  
      - Update `input_number.grid_export_meter` with last `2_8_0` state value  

18. **Connect all nodes accordingly** based on the described input/output relationships.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| # Subject & Attachment checks - Gmail payload uses `Subject` (capital S), IMAP node uses `subject` (lowercase). This template handles both cases. Attachment guard ensures `attachment_0` exists and its extension is `.xls` or `.xlsx`. Prerequisites include configuring Gmail and IMAP credentials properly.                                                                                                                                                                                                                                          | Sticky Note near Check Email Subject node                                                               |
| # Column Mapping - The E.ON export reuses column names with suffixes `_1`, `_2`, `_3`. Normalization maps these to standardized keys `start`, `AP`, `AM`, `1_8_0`, `2_8_0`. Three merge nodes combine data sets on `start`.                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note near Rename Keys and Merge nodes                                                             |
| # Time Conversion - Excel time is a serial day count added to base `1899-12-30`. Converted to ISO datetime then formatted to Spook-compatible string rounded to hour. Ensure n8n timezone matches HA or adjust accordingly.                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note near Convert Excel time and Convert datetime nodes                                           |
| # Hourly Grouping Logic (Code node) - Groups 15-min readings into hourly sums, resets meter baselines when new meter readings are present, initializes missing baselines at zero. Output objects contain `start`, `AP`, `AM`, `1_8_0`, and `2_8_0` fields formatted as strings with three decimals.                                                                                                                                                                                                                                                                                             | Sticky Note near Calculate hourly sum and node                                                          |
| # recorder.import_statistics payload - Constructs `{ start, state, sum }` arrays per meter. Service `recorder.import_statistics` updates long-term stats in Home Assistant Recorder via Spook integration.                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note near Generate 1_8_0 list for stats node                                                     |
| # Entity IDs updated - Uses entities `sensor.grid_energy_import` and `sensor.grid_energy_export` for stats, and `input_number.grid_import_meter` and `input_number.grid_export_meter` for last states. If entities are renamed, update these IDs in the relevant nodes. Template sensors in HA must have device_class: energy and state_class: total_increasing.                                                                                                                                                                                                                                            | Sticky Note near Home Assistant update nodes                                                            |
| # Credentials to configure - Gmail OAuth2 or IMAP read-only credentials, Home Assistant API with Long-Lived Access Token. Keep OAuth scopes minimal. Store secrets in n8n credentials, not node parameters.                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note near start of workflow                                                                       |
| # Troubleshooting - Common issues: no rows after extract (check subject & attachment), wrong times (timezone/excel serial), HA not showing stats (Recorder enabled?), 400/401 errors (token refresh), duplicate imports (hourly grouping is idempotent).                                                                                                                                                                                                                                                                                                                                             | Sticky Note near top left                                                                                 |
| # Security & Privacy - Use read-only email access, filter strictly by sender and subject, never hardcode tokens in nodes. Store credentials securely.                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note near Credentials notes                                                                       |
| # Home Assistant prerequisites - Recorder integration enabled (default if Energy dashboard used), Spook integration installed via HACS and HA restarted. Helpers `input_number.grid_import_meter` and `input_number.grid_export_meter` added for meter states. Template sensors for `grid_energy_import` and `grid_energy_export` created with appropriate device_class and state_class. See included YAML examples.                                                                                                                                            | Sticky Note with detailed HA setup instructions and links to helpers badge https://my.home-assistant.io/redirect/helpers/ |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively from an n8n automated workflow. This processing strictly respects content policies and contains no illegal or protected elements. All data handled is legal and public.