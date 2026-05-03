ZIP Code-Based Business Lead Generation with Google Maps and Sheets

https://n8nworkflows.xyz/workflows/zip-code-based-business-lead-generation-with-google-maps-and-sheets-11622


# ZIP Code-Based Business Lead Generation with Google Maps and Sheets

### 1. Workflow Overview

This workflow automates business lead generation based on ZIP codes, leveraging Google Maps API and Google Sheets for data management. It targets users who want to systematically collect and update business leads within specified ZIP codes and subcategories, ensuring data cleanliness and retry mechanisms for robustness.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception and Scheduling:** Periodic trigger to start the process and retrieve initial ZIP codes and subcategories from Google Sheets.
- **1.2 Data Filtering and Looping:** Filtering relevant subcategories and batching ZIP codes and subcategories for iterative processing.
- **1.3 Google Maps API Interaction:** Query Google Maps for businesses matching subcategories within ZIP codes, with logic to handle empty or failed responses.
- **1.4 Data Deduplication and Storage:** Remove duplicate leads and add new leads to Google Sheets.
- **1.5 Status Updates and Retry Mechanism:** Update processing status in Google Sheets, implement exponential backoff with wait nodes, and control retry attempts.
- **1.6 Notifications and Final Steps:** Notify via multiple channels (Telegram, Discord, Rapiwa) upon completion or errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

- **Overview:** This block initiates the workflow on schedule, fetching all ZIP codes and subcategories from Google Sheets to drive the lead generation.
- **Nodes Involved:** Schedule, Get All Zip Code, Limit, Get Subcategory form sheet, Filter subcategori, get subcategory
- **Node Details:**

  - **Schedule**
    - Type: Schedule Trigger
    - Role: Initiates workflow periodically
    - Config: Default scheduling (unspecified)
    - Inputs: None (trigger node)
    - Outputs: Connected to "Get All Zip Code"
    - Failures: None typical but check scheduling config
  - **Get All Zip Code**
    - Type: Google Sheets (Read)
    - Role: Retrieves all ZIP codes from a configured sheet
    - Config: Reads a specific sheet/range; executeOnce to avoid repeated reads
    - Inputs: From Schedule
    - Outputs: To Limit node
    - Failures: Google Sheets API limits or auth failure
  - **Limit**
    - Type: Limit
    - Role: Restricts number of ZIP codes processed per run (configurable)
    - Config: Limit count (not specified)
    - Inputs: From Get All Zip Code
    - Outputs: To Loop node
    - Failures: None expected
  - **Get Subcategory form sheet**
    - Type: Google Sheets (Read)
    - Role: Retrieves subcategory list from Google Sheets
    - Config: Reads a configured sheet/range; executeOnce enabled
    - Inputs: From Loop node (secondary path)
    - Outputs: To Filter subcategori node
    - Failures: API errors or auth failure
  - **Filter subcategori**
    - Type: Filter
    - Role: Filters subcategories based on criteria (criteria not specified)
    - Inputs: From Get Subcategory form sheet
    - Outputs: To get subcategory
    - Failures: Expression errors if conditions are invalid
  - **get subcategory**
    - Type: Set
    - Role: Prepares/sets subcategory data for downstream processing
    - Inputs: From Filter subcategori
    - Outputs: To Loop Subcats
    - Failures: None typical

---

#### 2.2 Data Filtering and Looping

- **Overview:** This block processes ZIP codes and subcategories in batches to manage API call volumes and parallelism.
- **Nodes Involved:** Loop, Loop Subcats, Set zip
- **Node Details:**

  - **Loop**
    - Type: SplitInBatches
    - Role: Splits ZIP codes into manageable batches
    - Config: Batch size (not specified)
    - Inputs: From Limit
    - Outputs: Secondary output to Get Subcategory form sheet (for chaining), primary to no output (empty branch)
    - Failures: Batch size misconfiguration could cause empty batches
  - **Loop Subcats**
    - Type: SplitInBatches
    - Role: Splits subcategory list into batches for processing
    - Config: Batch size (not specified)
    - Inputs: From get subcategory
    - Outputs: Two outputs: one to Loop (ZIP codes), another to Set zip
    - Failures: Similar to Loop
  - **Set zip**
    - Type: Set
    - Role: Sets current ZIP code value for subsequent API call
    - Inputs: From Loop Subcats
    - Outputs: To Google Map API node
    - Failures: Missing or invalid ZIP code data

---

#### 2.3 Google Maps API Interaction

- **Overview:** This critical block sends requests to Google Maps API to find businesses matching the ZIP code and subcategory, then decides next steps based on results.
- **Nodes Involved:** Google Map API, If, Place, Set Place
- **Node Details:**

  - **Google Map API**
    - Type: HTTP Request
    - Role: Calls Google Maps API with ZIP code and subcategory parameters
    - Config: Uses GET or POST with API key credentials; parameters built dynamically from Set zip and subcategory
    - Inputs: From Set zip
    - Outputs: To If node
    - Failures: API rate limit, auth errors, network timeouts
  - **If**
    - Type: If
    - Role: Checks if API returned valid data (e.g., results present)
    - Config: Conditional expression on API response presence or error
    - Inputs: From Google Map API
    - Outputs: True branch to Place node; False branch to Loop Subcats (retry or skip)
    - Failures: Expression evaluation errors
  - **Place**
    - Type: Code (JavaScript)
    - Role: Processes API response to extract or format place/business data
    - Config: Custom JS code to parse JSON and prepare lead data
    - Inputs: From If (true branch)
    - Outputs: To Set Place node
    - Failures: Parsing errors if API response format changes
  - **Set Place**
    - Type: Set
    - Role: Sets structured lead/place data for deduplication and storage
    - Inputs: From Place
    - Outputs: To Remove Duplicates node
    - Failures: Missing data fields or invalid data structures

---

#### 2.4 Data Deduplication and Storage

- **Overview:** Cleans lead data to remove duplicates and appends new leads to Google Sheets, then triggers status update.
- **Nodes Involved:** Remove Duplicates, Add rows in Google Sheets, Get Status, Update Status to Success
- **Node Details:**

  - **Remove Duplicates**
    - Type: Remove Duplicates
    - Role: Filters out duplicate leads based on defined keys (e.g., business name, address)
    - Config: Key fields set for uniqueness determination
    - Inputs: From Set Place
    - Outputs: To Add rows in Google Sheets
    - Failures: Incorrect keys causing false positives/negatives
  - **Add rows in Google Sheets**
    - Type: Google Sheets (Append)
    - Role: Adds new leads as rows to a configured Google Sheet
    - Config: Sheet ID, range, and append mode
    - Inputs: From Remove Duplicates
    - Outputs: Two branches: one to Get Status, one to Exponential Backoff (retry logic)
    - Failures: Google API errors, rate limits, invalid data
  - **Get Status**
    - Type: Google Sheets (Read)
    - Role: Reads current processing status from sheet to decide further flow
    - Config: Reads specific cell/range; executeOnce enabled
    - Inputs: From Add rows in Google Sheets
    - Outputs: To Update Status to Success and Do nothing node
    - Failures: API errors or empty data
  - **Update Status to Success**
    - Type: Google Sheets (Update)
    - Role: Updates status cell/range to indicate success
    - Config: Target cell, value set to “Success” or equivalent
    - Inputs: From Get Status and Check maximum retries nodes
    - Outputs: To Loop Subcats and Exponential Backoff1 for next cycle or wait
    - Failures: API write errors

---

#### 2.5 Status Updates and Retry Mechanism

- **Overview:** Implements a retry mechanism with exponential backoff and wait nodes to handle API limits or failures gracefully.
- **Nodes Involved:** Exponential Backoff, Exponential Backoff1, Wait, Wait1, Check Max Retries, Check maximum retries., Do nothing
- **Node Details:**

  - **Exponential Backoff**
    - Type: Code
    - Role: Calculates delay time dynamically to increase wait on consecutive failures
    - Config: Custom JS code implementing backoff algorithm
    - Inputs: From Add rows in Google Sheets (failure path)
    - Outputs: To Wait node
    - Failures: Logic errors in code
  - **Wait**
    - Type: Wait
    - Role: Pauses workflow for calculated backoff period
    - Inputs: From Exponential Backoff
    - Outputs: To Check Max Retries
    - Failures: Misconfigured wait times
  - **Check Max Retries**
    - Type: If
    - Role: Checks if retry count exceeded max allowed attempts
    - Inputs: From Wait
    - Outputs: True to Do nothing (stop retries), False to Add rows in Google Sheets (retry)
    - Failures: Expression errors
  - **Exponential Backoff1**
    - Type: Code
    - Role: Similar to Exponential Backoff but for status update retries
    - Inputs: From Update Status to Success
    - Outputs: To Wait1 node
    - Failures: Logic errors
  - **Wait1**
    - Type: Wait
    - Role: Wait node for status update retries
    - Inputs: From Exponential Backoff1
    - Outputs: To Check maximum retries.
    - Failures: Misconfiguration
  - **Check maximum retries.**
    - Type: If
    - Role: Checks retry count for status update
    - Inputs: From Wait1
    - Outputs: True branch to Do nothing, False branch to Update Status to Success
    - Failures: Expression errors
  - **Do nothing**
    - Type: No Operation
    - Role: Terminal node to stop retry loops or proceed
    - Inputs: Multiple (from retry checks)
    - Outputs: To multiple notification nodes (Rapiwa, Telegram, Discord)
    - Failures: None

---

#### 2.6 Notifications and Final Steps

- **Overview:** Sends notifications via Telegram, Discord, and Rapiwa to signal workflow completion or issues.
- **Nodes Involved:** Rapiwa, Send a text message (Telegram), Send a message (Discord)
- **Node Details:**

  - **Rapiwa**
    - Type: Rapiwa node (third-party notification)
    - Role: Sends notification messages
    - Config: Requires Rapiwa credentials and message parameters
    - Inputs: From Do nothing
    - Outputs: None
    - Failures: Auth errors or API limits
  - **Send a text message**
    - Type: Telegram
    - Role: Sends Telegram messages
    - Config: Uses Telegram OAuth2 credentials and chat ID/message content
    - Inputs: From Do nothing
    - Outputs: None
    - Failures: Telegram API errors, invalid chat ID
  - **Send a message**
    - Type: Discord
    - Role: Sends messages to Discord channels via webhook
    - Config: Discord webhook URL and message content
    - Inputs: From Do nothing
    - Outputs: None
    - Failures: Webhook errors or rate limits

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                         | Input Node(s)                 | Output Node(s)                          | Sticky Note                                  |
|-------------------------|----------------------------|---------------------------------------|------------------------------|----------------------------------------|----------------------------------------------|
| Schedule                | Schedule Trigger           | Start workflow periodically           | None                         | Get All Zip Code                       |                                              |
| Get All Zip Code         | Google Sheets (Read)       | Get all ZIP codes                     | Schedule                     | Limit                                 |                                              |
| Limit                   | Limit                      | Restrict number of ZIP codes          | Get All Zip Code             | Loop                                  |                                              |
| Loop                    | SplitInBatches             | Batch processing of ZIP codes         | Limit                       | Get Subcategory form sheet (secondary) |                                              |
| Get Subcategory form sheet | Google Sheets (Read)     | Get subcategories                     | Loop (secondary output)      | Filter subcategori                    |                                              |
| Filter subcategori       | Filter                     | Filter relevant subcategories          | Get Subcategory form sheet   | get subcategory                       |                                              |
| get subcategory          | Set                        | Prepare subcategory data               | Filter subcategori           | Loop Subcats                         |                                              |
| Loop Subcats             | SplitInBatches             | Batch processing of subcategories     | get subcategory              | Loop, Set zip                        |                                              |
| Set zip                 | Set                        | Set ZIP code for API query             | Loop Subcats                 | Google Map API                       |                                              |
| Google Map API           | HTTP Request               | Query Google Maps API                   | Set zip                     | If                                   |                                              |
| If                      | If                         | Check API response validity            | Google Map API              | Place (true), Loop Subcats (false)  |                                              |
| Place                   | Code                       | Parse and format place data            | If (true)                   | Set Place                           |                                              |
| Set Place                | Set                        | Prepare place data                     | Place                       | Remove Duplicates                   |                                              |
| Remove Duplicates        | Remove Duplicates          | Remove duplicate leads                 | Set Place                   | Add rows in Google Sheets           |                                              |
| Add rows in Google Sheets | Google Sheets (Append)    | Append leads to sheet                  | Remove Duplicates            | Get Status, Exponential Backoff      |                                              |
| Get Status               | Google Sheets (Read)       | Read status from sheet                 | Add rows in Google Sheets    | Update Status to Success, Do nothing |                                              |
| Update Status to Success | Google Sheets (Update)     | Update status to success               | Get Status, Check maximum retries. | Loop Subcats, Exponential Backoff1 |                                              |
| Exponential Backoff      | Code                       | Calculate delay for retry              | Add rows in Google Sheets    | Wait                               |                                              |
| Wait                    | Wait                       | Wait before retry                      | Exponential Backoff          | Check Max Retries                   |                                              |
| Check Max Retries        | If                         | Check retry attempts limit             | Wait                        | Do nothing (max reached), Add rows (retry) |                                              |
| Exponential Backoff1     | Code                       | Delay calculation for status update retry | Update Status to Success      | Wait1                              |                                              |
| Wait1                   | Wait                       | Wait before status retry               | Exponential Backoff1         | Check maximum retries.              |                                              |
| Check maximum retries.   | If                         | Check retry limit for status update   | Wait1                       | Do nothing (max), Update Status to Success (retry) |                                              |
| Do nothing              | No Operation               | Terminal/no-op node                    | Get Status, Check Max Retries, Check maximum retries. | Rapiwa, Send a text message, Send a message |                                              |
| Rapiwa                  | Rapiwa (Notification)      | Send notification                      | Do nothing                  | None                               |                                              |
| Send a text message      | Telegram                   | Send Telegram notification             | Do nothing                  | None                               |                                              |
| Send a message           | Discord                    | Send Discord notification              | Do nothing                  | None                               |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run on desired periodicity (e.g., daily, hourly).

2. **Add Google Sheets Node: Get All Zip Code**  
   - Type: Google Sheets (Read)  
   - Connect from Schedule node.  
   - Configure with sheet ID and range containing ZIP codes.  
   - Enable "Execute Once" to avoid repeated reads.

3. **Add Limit Node**  
   - Type: Limit  
   - Connect from Get All Zip Code.  
   - Set maximum number of ZIP codes to process per run.

4. **Add SplitInBatches Node: Loop**  
   - Type: SplitInBatches  
   - Connect from Limit node.  
   - Configure batch size for ZIP code processing.

5. **Add Google Sheets Node: Get Subcategory form sheet**  
   - Type: Google Sheets (Read)  
   - Connect secondary output of Loop node.  
   - Configure with sheet containing subcategories.  
   - Enable "Execute Once".

6. **Add Filter Node: Filter subcategori**  
   - Type: Filter  
   - Connect from Get Subcategory form sheet.  
   - Configure filter expression(s) for selecting valid subcategories.

7. **Add Set Node: get subcategory**  
   - Type: Set  
   - Connect from Filter subcategori.  
   - Prepare variables or fields for looping.

8. **Add SplitInBatches Node: Loop Subcats**  
   - Type: SplitInBatches  
   - Connect from get subcategory.  
   - Configure batch size for subcategory processing.

9. **Add Set Node: Set zip**  
   - Type: Set  
   - Connect from Loop Subcats (secondary output).  
   - Map current ZIP code variable for API query.

10. **Add HTTP Request Node: Google Map API**  
    - Type: HTTP Request  
    - Connect from Set zip.  
    - Configure with Google Maps API endpoint, parameters including ZIP code and subcategory, and API credentials.

11. **Add If Node**  
    - Type: If  
    - Connect from Google Map API.  
    - Configure condition to check if API response contains valid results.

12. **Add Code Node: Place**  
    - Type: Code  
    - Connect true output of If node.  
    - Implement JavaScript to parse and format API response into lead data.

13. **Add Set Node: Set Place**  
    - Type: Set  
    - Connect from Place node.  
    - Format and prepare data for deduplication.

14. **Add Remove Duplicates Node**  
    - Type: Remove Duplicates  
    - Connect from Set Place.  
    - Configure keys for duplicate detection (e.g., business name, address).

15. **Add Google Sheets Node: Add rows in Google Sheets**  
    - Type: Google Sheets (Append)  
    - Connect from Remove Duplicates.  
    - Configure target sheet and range for appending leads.

16. **Add Google Sheets Node: Get Status**  
    - Type: Google Sheets (Read)  
    - Connect from Add rows in Google Sheets.  
    - Configure to read processing status cell/range.  
    - Enable "Execute Once".

17. **Add Google Sheets Node: Update Status to Success**  
    - Type: Google Sheets (Update)  
    - Connect from Get Status.  
    - Configure to write “Success” or equivalent status.

18. **Add Code Node: Exponential Backoff**  
    - Type: Code  
    - Connect failure output of Add rows in Google Sheets.  
    - Implement retry delay logic.

19. **Add Wait Node**  
    - Type: Wait  
    - Connect from Exponential Backoff.  
    - Use delay calculated by backoff.

20. **Add If Node: Check Max Retries**  
    - Type: If  
    - Connect from Wait node.  
    - Configure to compare retry count against max allowed.

21. **Add NoOp Node: Do nothing**  
    - Type: No Operation  
    - Connect true output of Check Max Retries (stop retries).  
    - Connect false output to Add rows in Google Sheets (retry).

22. **Add Code Node: Exponential Backoff1**  
    - Type: Code  
    - Connect from Update Status to Success.  
    - Similar retry delay logic for status updates.

23. **Add Wait Node: Wait1**  
    - Type: Wait  
    - Connect from Exponential Backoff1.

24. **Add If Node: Check maximum retries.**  
    - Type: If  
    - Connect from Wait1.  
    - Retry limit check for status update retries.

25. **Connect true output of Check maximum retries. to Do nothing node**  
    - Stop retries if max reached.

26. **Connect false output to Update Status to Success**  
    - Retry status update.

27. **Add Notification Nodes:**  
    - **Rapiwa** (requires credentials and setup)  
    - **Telegram Send a text message** (OAuth2 credentials, chat ID)  
    - **Discord Send a message** (webhook URL)  
    - Connect all from Do nothing node.

28. **Configure Credentials:**  
    - Google Sheets OAuth2 or API key with edit permissions  
    - Google Maps API key with Places API enabled  
    - Telegram Bot Token and Chat ID  
    - Discord Webhook URL  
    - Rapiwa credentials (if used)

29. **Test with smaller batch sizes and controlled data to ensure flow correctness.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                             |
|-------------------------------------------------------------------------------------------------|--------------------------------------------|
| Workflow automates lead generation via ZIP codes and business subcategories using Google Maps API. | Project description                         |
| Includes exponential backoff strategy to handle API rate limits and failure recovery.           | Reliability and fault tolerance             |
| Notifications via Telegram, Discord, and Rapiwa for real-time alerts on workflow status.        | Multi-channel notification setup            |
| Google Sheets serve as both input source and final storage for leads and status tracking.       | Centralized data management                  |
| Ensure Google Maps API billing is enabled to avoid request failures due to quota limits.        | Google Cloud Console setup                    |
| For Telegram notifications, create bot and obtain chat ID for message delivery.                 | https://core.telegram.org/bots/api          |
| Discord webhook URL required for Discord node configuration.                                    | https://discord.com/developers/docs/resources/webhook |
| Rapiwa usage requires account and API key setup.                                                | https://rapiwa.com                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. All data processed complies with legal and public content policies. No illegal, offensive, or protected content is involved.