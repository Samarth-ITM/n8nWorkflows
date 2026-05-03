Enrich Procurement Contacts from Company IDs with Apollo, Google Sheets & Telegram

https://n8nworkflows.xyz/workflows/enrich-procurement-contacts-from-company-ids-with-apollo--google-sheets---telegram-7589


# Enrich Procurement Contacts from Company IDs with Apollo, Google Sheets & Telegram

### 1. Workflow Overview

This workflow automates the enrichment of procurement contact data by leveraging company IDs and integrating Apollo’s API, Google Sheets, and Telegram notifications. It is designed for users who maintain company data in Google Sheets and want to enhance contact information automatically by querying Apollo’s database. The workflow also handles incoming leads via webhook and updates relevant sheets while notifying users of successes or errors.

The workflow logic is divided into the following blocks:

- **1.1 Scheduled Data Retrieval:** Periodically triggers the process to fetch company IDs from Google Sheets.
- **1.2 Search Preparation:** Defines search parameters and builds filters for Apollo API requests.
- **1.3 Apollo People Search & Processing:** Queries Apollo for people related to company IDs and processes each found contact.
- **1.4 Data Persistence:** Saves enriched contact data and marks companies as processed in Google Sheets.
- **1.5 Lead Reception:** Handles incoming lead data via webhook, saves phone numbers to Google Sheets, and sends success notifications.
- **1.6 Error Handling:** Catches workflow errors, formats messages, and sends alerts to Telegram.
- **1.7 Control & Timing:** Implements pauses and controls scheduling to avoid API rate limits or overload.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

- **Overview:** Periodically triggers the workflow and retrieves company IDs from Google Sheets to start enrichment.
- **Nodes Involved:** `Run Every X Minutes`, `Get Apollo Company ID`
- **Node Details:**

  - **Run Every X Minutes**
    - Type: Schedule Trigger
    - Role: Initiates workflow execution regularly based on a timer.
    - Configuration: Default schedule settings (likely every few minutes).
    - Inputs: None
    - Outputs: Triggers `Get Apollo Company ID`
    - Potential Failures: Scheduling misconfiguration, node disabled.
  
  - **Get Apollo Company ID**
    - Type: Google Sheets
    - Role: Reads company IDs from a configured Google Sheet.
    - Configuration: Connects via Google OAuth2 credentials; configured to read a specific sheet/range containing company IDs to process.
    - Inputs: Trigger from schedule.
    - Outputs: Data to `Define Search Settings`
    - Potential Failures: Google Sheets API errors, authentication failure, empty or malformed data.

#### 2.2 Search Preparation

- **Overview:** Prepares the parameters and filters required for searching contacts on Apollo.
- **Nodes Involved:** `Define Search Settings`, `Build Search Filters`
- **Node Details:**

  - **Define Search Settings**
    - Type: Set
    - Role: Initializes or sets fixed search parameters, possibly including API keys, filter templates, or static values.
    - Configuration: Sets key-value pairs used later in API requests.
    - Inputs: Data from `Get Apollo Company ID`
    - Outputs: Passes to `Build Search Filters`
    - Potential Failures: Missing or incorrect parameter values leading to failed API calls.

  - **Build Search Filters**
    - Type: Code (JavaScript)
    - Role: Constructs complex filter objects or query strings dynamically based on input data.
    - Configuration: Executes JavaScript code to transform input data into Apollo-compatible search filters.
    - Inputs: From `Define Search Settings`
    - Outputs: Filtered data to `Search People in Apollo`
    - Key Expressions: Likely involves JSON parsing/stringifying, filtering arrays, or mapping data.
    - Potential Failures: Code exceptions, invalid filter syntax.

#### 2.3 Apollo People Search & Processing

- **Overview:** Executes Apollo API calls to search for people linked to company IDs and processes each person individually.
- **Nodes Involved:** `Search People in Apollo`, `Check if Any People Found`, `Process Each Person One by One`, `Get Person Contact Details`
- **Node Details:**

  - **Search People in Apollo**
    - Type: HTTP Request
    - Role: Calls Apollo’s people search API with filters.
    - Configuration: HTTP method POST/GET, headers with API key, URL endpoint for Apollo people search.
    - Inputs: Filters from `Build Search Filters`
    - Outputs: API response to `Check if Any People Found`
    - Potential Failures: API authentication errors, rate limiting, network timeouts.

  - **Check if Any People Found**
    - Type: If
    - Role: Conditional branching to check if results exist.
    - Configuration: Evaluates if the API response contains any people records.
    - Inputs: Apollo response
    - Outputs: 
      - If true: `Process Each Person One by One`
      - If false: `Mark Company as Processed (No Person Found)`
    - Potential Failures: Improper condition expression, unexpected API response format.

  - **Process Each Person One by One**
    - Type: Split Out
    - Role: Splits array of people into individual items for sequential processing.
    - Configuration: Splits data for one-by-one processing.
    - Inputs: People array from `Check if Any People Found`
    - Outputs: Each person to `Get Person Contact Details`
    - Potential Failures: Empty input array, splitting errors.

  - **Get Person Contact Details**
    - Type: HTTP Request
    - Role: Retrieves detailed contact information for each person from Apollo.
    - Configuration: API call per person using their unique ID.
    - Inputs: Single person data from splitter
    - Outputs: Detailed contact data to `Save Person to Google Sheet`
    - Potential Failures: API authentication, rate limiting, missing person ID.

#### 2.4 Data Persistence

- **Overview:** Saves enriched contact details and updates Google Sheets to track processing status.
- **Nodes Involved:** `Save Person to Google Sheet`, `Mark Company as Processed`, `Mark Company as Processed (No Person Found)`, `Pause Before Next Run`
- **Node Details:**

  - **Save Person to Google Sheet**
    - Type: Google Sheets
    - Role: Appends or updates rows with detailed person contact data.
    - Configuration: Connected to Google Sheets, configured to write enriched data into a designated sheet.
    - Inputs: Contact details from `Get Person Contact Details`
    - Outputs: Triggers `Mark Company as Processed`
    - Potential Failures: Google API limits, invalid data format.

  - **Mark Company as Processed**
    - Type: Google Sheets
    - Role: Marks the company ID as processed after successful enrichment.
    - Configuration: Updates a status column or flag in the companies sheet.
    - Inputs: After saving person data.
    - Outputs: Possibly ends the processing chain for this company.
    - Potential Failures: Write conflicts, API errors.

  - **Mark Company as Processed (No Person Found)**
    - Type: Google Sheets
    - Role: Marks companies with no people found to avoid reprocessing.
    - Configuration: Updates status in sheet accordingly.
    - Inputs: From `Check if Any People Found` false branch.
    - Outputs: Goes to `Pause Before Next Run`
    - Potential Failures: Same as above.

  - **Pause Before Next Run**
    - Type: Wait
    - Role: Adds a delay before next scheduled run, helping with API rate limits or pacing.
    - Configuration: Fixed wait duration.
    - Inputs: From marking nodes.
    - Outputs: Loops back to `Get Apollo Company ID`
    - Potential Failures: None significant unless workflow interrupted.

#### 2.5 Lead Reception

- **Overview:** Receives leads via webhook, saves phone numbers to Google Sheets, and sends Telegram notifications.
- **Nodes Involved:** `Receive Lead (Group 1)`, `Save Lead Phone to Sheet`, `Notify: Success`
- **Node Details:**

  - **Receive Lead (Group 1)**
    - Type: Webhook
    - Role: Entry point for receiving lead data from external sources.
    - Configuration: Configured with unique webhook URL.
    - Inputs: HTTP POST data with lead information.
    - Outputs: Leads data to `Save Lead Phone to Sheet`
    - Potential Failures: Webhook misconfiguration, invalid payload.

  - **Save Lead Phone to Sheet**
    - Type: Google Sheets
    - Role: Saves incoming lead phone numbers to a dedicated sheet.
    - Configuration: Includes retry logic with 2 max tries and 5s delay.
    - Inputs: From webhook.
    - Outputs: Triggers `Notify: Success`
    - Potential Failures: Google API errors, retries on failure.

  - **Notify: Success**
    - Type: Telegram
    - Role: Sends a Telegram message confirming lead saved successfully.
    - Configuration: Uses Telegram bot credentials, sends to configured chat ID.
    - Inputs: Success signal from sheet save.
    - Outputs: None
    - Potential Failures: Telegram API issues, credential problems.

#### 2.6 Error Handling

- **Overview:** Captures workflow errors, formats error messages, and sends alerts to Telegram.
- **Nodes Involved:** `Catch Workflow Error`, `Format Error Message`, `Send Alert to Telegram`
- **Node Details:**

  - **Catch Workflow Error**
    - Type: Error Trigger
    - Role: Globally captures any errors occurring in the workflow.
    - Configuration: Listens on all workflow nodes.
    - Inputs: Errors automatically caught.
    - Outputs: Sends error info to `Format Error Message`
    - Potential Failures: None (designed for error capture).

  - **Format Error Message**
    - Type: Code (JavaScript)
    - Role: Formats error data into readable message text for alerts.
    - Configuration: Custom JavaScript formatting error details.
    - Inputs: Raw error data.
    - Outputs: Formatted message to `Send Alert to Telegram`
    - Potential Failures: Code exceptions.

  - **Send Alert to Telegram**
    - Type: Telegram
    - Role: Sends formatted error messages to a Telegram chat.
    - Configuration: Uses Telegram Bot credentials and chat id.
    - Inputs: Formatted error message.
    - Outputs: None.
    - Potential Failures: Telegram connectivity, credential issues.

---

### 3. Summary Table

| Node Name                        | Node Type          | Functional Role                              | Input Node(s)                     | Output Node(s)                                 | Sticky Note |
|---------------------------------|--------------------|----------------------------------------------|----------------------------------|------------------------------------------------|-------------|
| Run Every X Minutes              | Schedule Trigger   | Triggers periodic workflow runs               | None                             | Get Apollo Company ID                          |             |
| Get Apollo Company ID            | Google Sheets      | Reads company IDs to process                   | Run Every X Minutes              | Define Search Settings                         |             |
| Define Search Settings           | Set                | Sets static/dynamic search parameters          | Get Apollo Company ID            | Build Search Filters                           |             |
| Build Search Filters             | Code               | Builds Apollo API search filters               | Define Search Settings           | Search People in Apollo                        |             |
| Search People in Apollo          | HTTP Request       | Calls Apollo API to search people               | Build Search Filters             | Check if Any People Found                      |             |
| Check if Any People Found        | If                 | Branches based on search results                | Search People in Apollo          | Process Each Person One by One / Mark Company as Processed (No Person Found) |             |
| Process Each Person One by One   | Split Out          | Splits people array for individual processing  | Check if Any People Found        | Get Person Contact Details                     |             |
| Get Person Contact Details       | HTTP Request       | Retrieves detailed person contact info          | Process Each Person One by One   | Save Person to Google Sheet                    |             |
| Save Person to Google Sheet      | Google Sheets      | Saves enriched contact details                   | Get Person Contact Details       | Mark Company as Processed                      |             |
| Mark Company as Processed        | Google Sheets      | Marks company as processed                        | Save Person to Google Sheet      | None                                           |             |
| Mark Company as Processed (No Person Found) | Google Sheets | Marks company processed when no contacts found  | Check if Any People Found (false branch) | Pause Before Next Run                |             |
| Pause Before Next Run            | Wait               | Pauses workflow to manage rate limits            | Mark Company as Processed (No Person Found) | Get Apollo Company ID                 |             |
| Receive Lead (Group 1)           | Webhook            | Receives lead data                                | External HTTP                   | Save Lead Phone to Sheet                       |             |
| Save Lead Phone to Sheet         | Google Sheets      | Saves lead phone numbers                           | Receive Lead (Group 1)           | Notify: Success                               |             |
| Notify: Success                 | Telegram            | Sends success notification for lead saved        | Save Lead Phone to Sheet         | None                                           |             |
| Catch Workflow Error             | Error Trigger      | Captures workflow errors                           | Automatic                      | Format Error Message                          |             |
| Format Error Message             | Code               | Formats error messages for alerts                  | Catch Workflow Error             | Send Alert to Telegram                         |             |
| Send Alert to Telegram           | Telegram            | Sends error alerts                                 | Format Error Message             | None                                           |             |
| Sticky Note                     | Sticky Note         | Visual notes (not functionally connected)         | None                           | None                                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Name: `Run Every X Minutes`
   - Type: Schedule Trigger
   - Configuration: Set interval to desired minutes (e.g., every 5 minutes).

2. **Add Google Sheets Node for Company IDs**
   - Name: `Get Apollo Company ID`
   - Type: Google Sheets
   - Credentials: Connect Google OAuth2 credentials with access to your company IDs sheet.
   - Configure to read your company IDs (sheet name, range).
   - Connect trigger output from `Run Every X Minutes` to this node.

3. **Add Set Node for Search Parameters**
   - Name: `Define Search Settings`
   - Type: Set
   - Set static or dynamic parameters required for Apollo search (e.g., API keys, search options).
   - Connect from `Get Apollo Company ID`.

4. **Add Code Node to Build Filters**
   - Name: `Build Search Filters`
   - Type: Code (JavaScript)
   - Write code to construct Apollo search filters based on input data.
   - Connect from `Define Search Settings`.

5. **Add HTTP Request Node to Search People in Apollo**
   - Name: `Search People in Apollo`
   - Type: HTTP Request
   - Method: POST or GET (as per Apollo API)
   - URL: Apollo people search endpoint
   - Headers: Include Apollo API key (set in credentials)
   - Body: Use filters from `Build Search Filters`
   - Connect from `Build Search Filters`.

6. **Add If Node to Check Results**
   - Name: `Check if Any People Found`
   - Type: If
   - Condition: Check if response body contains people records.
   - True branch: Connect to `Process Each Person One by One`
   - False branch: Connect to `Mark Company as Processed (No Person Found)`
   - Connect from `Search People in Apollo`.

7. **Add Split Out Node**
   - Name: `Process Each Person One by One`
   - Type: Split Out
   - Configuration: Split array of people into single items.
   - Connect from true branch of `Check if Any People Found`.

8. **Add HTTP Request Node for Person Details**
   - Name: `Get Person Contact Details`
   - Type: HTTP Request
   - Configure to call Apollo API for detailed person info using person ID.
   - Connect from `Process Each Person One by One`.

9. **Add Google Sheets Node to Save Person**
   - Name: `Save Person to Google Sheet`
   - Type: Google Sheets
   - Configure to append contact details to your sheet.
   - Connect from `Get Person Contact Details`.

10. **Add Google Sheets Node to Mark Company Processed**
    - Name: `Mark Company as Processed`
    - Type: Google Sheets
    - Configure to update status column for processed company ID.
    - Connect from `Save Person to Google Sheet`.

11. **Add Google Sheets Node for No Person Found**
    - Name: `Mark Company as Processed (No Person Found)`
    - Type: Google Sheets
    - Configure to mark company as processed even if no person found.
    - Connect from false branch of `Check if Any People Found`.

12. **Add Wait Node for Rate Limiting**
    - Name: `Pause Before Next Run`
    - Type: Wait
    - Set delay duration (e.g., 1 minute).
    - Connect from `Mark Company as Processed (No Person Found)`.

13. **Loop Back**
    - Connect output of `Pause Before Next Run` to `Get Apollo Company ID` to enable continuous runs.

14. **Add Webhook Node for Incoming Leads**
    - Name: `Receive Lead (Group 1)`
    - Type: Webhook
    - Configure with unique webhook URL to accept HTTP POST lead data.

15. **Add Google Sheets Node to Save Lead Phone**
    - Name: `Save Lead Phone to Sheet`
    - Type: Google Sheets
    - Configure to append lead phone information.
    - Enable retry with 2 attempts and 5-second delay.
    - Connect from `Receive Lead (Group 1)`.

16. **Add Telegram Node for Success Notification**
    - Name: `Notify: Success`
    - Type: Telegram
    - Configure Telegram Bot credentials and chat ID.
    - Connect from `Save Lead Phone to Sheet`.

17. **Add Error Trigger Node**
    - Name: `Catch Workflow Error`
    - Type: Error Trigger
    - Captures any errors during workflow execution.

18. **Add Code Node to Format Errors**
    - Name: `Format Error Message`
    - Type: Code (JavaScript)
    - Write code to format error details into a user-friendly message.
    - Connect from `Catch Workflow Error`.

19. **Add Telegram Node for Error Alerts**
    - Name: `Send Alert to Telegram`
    - Type: Telegram
    - Configure to send error messages to a Telegram chat.
    - Connect from `Format Error Message`.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                   |
|--------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Apollo API requires valid API keys and proper usage of endpoints; check https://www.apollo.io/docs for latest API docs. | Apollo API documentation                          |
| Google Sheets credentials must have read/write access to the specified spreadsheets.              | Google Sheets API setup                           |
| Telegram Bot token and chat ID must be configured correctly to receive messages.                  | Telegram Bot API guide: https://core.telegram.org/bots/api |
| Workflow uses retry logic on Google Sheets node for lead saving to handle transient errors.       | n8n retry mechanism documentation                 |
| Proper error handling ensures alerts on failures for timely intervention.                         | n8n error trigger node documentation              |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow and strictly adheres to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.