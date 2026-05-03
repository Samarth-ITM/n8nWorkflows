Find mobile numbers with ScraperCity from LinkedIn and notify sales in Slack

https://n8nworkflows.xyz/workflows/find-mobile-numbers-with-scrapercity-from-linkedin-and-notify-sales-in-slack-14383


# Find mobile numbers with ScraperCity from LinkedIn and notify sales in Slack

# 1. Workflow Overview

This workflow manually submits a batch of LinkedIn profile URLs and work email addresses to the ScraperCity mobile-finder API, waits for the asynchronous job to complete, downloads the resulting contact file, filters for records containing phone numbers, and sends a formatted alert to a Slack sales channel.

Its primary use case is sales enrichment: a user provides a list of prospect identifiers, ScraperCity attempts to find mobile numbers, and the workflow posts the successful matches to Slack for follow-up.

## 1.1 Manual Input Reception and Configuration
This block starts the workflow manually and defines the three runtime inputs:
- LinkedIn profile URLs
- Work emails
- Target Slack channel

## 1.2 API Job Construction and Submission
This block transforms the configured comma-separated inputs into a single array and submits them to ScraperCity as a mobile-finder job. It then stores the returned run ID for later polling and download.

## 1.3 Initial Wait and Polling Loop
Because ScraperCity processes the request asynchronously, the workflow pauses before checking status, then enters a loop structure that repeatedly queries the job state until it is marked complete.

## 1.4 Status Decision and Retry Handling
This block checks the current ScraperCity job status. If complete, it proceeds to results download; otherwise, it waits and loops back for another status check.

## 1.5 Results Download and Parsing
Once the job finishes, the workflow downloads the results file, parses CSV-like content into one item per contact, and removes duplicate records.

## 1.6 Contact Filtering and Slack Message Preparation
This block keeps only contacts that appear to contain a mobile or phone field, then aggregates all surviving records into one Slack-formatted message.

## 1.7 Slack Notification Delivery
The final block posts the formatted message to the configured Slack channel to alert the sales team.

---

# 2. Block-by-Block Analysis

## 2.1 Manual Input Reception and Configuration

**Overview:**  
This block provides the workflow entry point and defines the static/manual inputs used during execution. It is the only entry node in the workflow and must be run manually.

**Nodes Involved:**  
- When clicking 'Execute workflow'
- Configure Inputs

### Node: When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual workflow trigger. Starts execution only when a user clicks Execute.
- **Configuration choices:**  
  No custom parameters are set.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Configure Inputs`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`; standard manual trigger behavior.
- **Edge cases or potential failure types:**  
  - No automation scheduling; must be run manually
  - Not suitable as-is for production unattended execution
- **Sub-workflow reference:**  
  None.

### Node: Configure Inputs
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the runtime configuration payload.
- **Configuration choices:**  
  Defines three string fields:
  - `linkedinProfiles`: comma-separated LinkedIn profile URLs
  - `workEmails`: comma-separated work emails
  - `slackChannel`: Slack channel name such as `#sales-alerts`
- **Key expressions or variables used:**  
  Static values are currently hard-coded in the node.
- **Input and output connections:**  
  - Input: `When clicking 'Execute workflow'`
  - Output: `Build Inputs Array`
- **Version-specific requirements:**  
  Uses `Set` node version `3.4`.
- **Edge cases or potential failure types:**  
  - Empty strings lead to empty inputs later
  - Mismatch between LinkedIn URLs and emails is not validated
  - Slack channel may fail later if the bot cannot resolve channel name
- **Sub-workflow reference:**  
  None.

---

## 2.2 API Job Construction and Submission

**Overview:**  
This block normalizes the input strings into arrays, combines them into the format required by ScraperCity, submits the enrichment job, and preserves the returned run ID plus Slack channel context.

**Nodes Involved:**  
- Build Inputs Array
- Submit Mobile Finder Job
- Store Run ID

### Node: Build Inputs Array
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses comma-separated input fields into a single `inputs` array.
- **Configuration choices:**  
  The JavaScript code:
  - reads `linkedinProfiles` and `workEmails`
  - splits each by comma
  - trims whitespace
  - removes empty elements
  - concatenates the two arrays
  - passes through `slackChannel`
- **Key expressions or variables used:**  
  - `$input.first().json.linkedinProfiles`
  - `$input.first().json.workEmails`
  - `$input.first().json.slackChannel`
- **Input and output connections:**  
  - Input: `Configure Inputs`
  - Output: `Submit Mobile Finder Job`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - If all values are empty, `inputs` becomes an empty array
  - No validation for URL format or email format
  - No pairing logic between LinkedIn URLs and emails; values are simply merged
- **Sub-workflow reference:**  
  None.

### Node: Submit Mobile Finder Job
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the job creation request to ScraperCity.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/mobile-finder`
  - Authentication: generic credential type using HTTP Header Auth
  - Body type: JSON
  - Request body sends `{ inputs: $json.inputs }`
- **Key expressions or variables used:**  
  - `={{ JSON.stringify({ inputs: $json.inputs }) }}`
- **Input and output connections:**  
  - Input: `Build Inputs Array`
  - Output: `Store Run ID`
- **Version-specific requirements:**  
  HTTP Request node version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid API key / header auth failure
  - ScraperCity endpoint mismatch if API version changes
  - Empty `inputs` may be rejected by ScraperCity
  - Unexpected response shape could break downstream `runId` extraction
  - Network timeout or non-2xx response
- **Sub-workflow reference:**  
  None.

### Node: Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores the returned job ID and reattaches the Slack channel for later use.
- **Configuration choices:**  
  Creates:
  - `runId` from the API response
  - `slackChannel` from `Build Inputs Array`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
  - `={{ $('Build Inputs Array').item.json.slackChannel }}`
- **Input and output connections:**  
  - Input: `Submit Mobile Finder Job`
  - Output: `Wait 30 Seconds Before First Poll`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - If ScraperCity returns a different field than `runId`, polling will fail
  - Cross-node expression depends on `Build Inputs Array` being present in execution context
- **Sub-workflow reference:**  
  None.

---

## 2.3 Initial Wait and Polling Loop

**Overview:**  
This block introduces an initial delay before polling and establishes the loop mechanism used to repeatedly check job status.

**Nodes Involved:**  
- Wait 30 Seconds Before First Poll
- Poll Loop Controller

### Node: Wait 30 Seconds Before First Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays the first status check so ScraperCity has time to begin processing.
- **Configuration choices:**  
  Wait amount is set to `30` seconds.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Store Run ID`
  - Output: `Poll Loop Controller`
- **Version-specific requirements:**  
  Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - Too short a wait may increase unnecessary polling
  - Very long waits delay notification unnecessarily
- **Sub-workflow reference:**  
  None.

### Node: Poll Loop Controller
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Used here as a loop-control mechanism rather than for actual batching.
- **Configuration choices:**  
  - `batchSize: 1`
  - `reset: false`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Input: `Wait 30 Seconds Before First Poll`
  - Output: `Check Job Status`
  - Additional loop-back input from `Wait 60 Seconds Before Retry`
- **Version-specific requirements:**  
  Split In Batches node version `3`.
- **Edge cases or potential failure types:**  
  - This is a workaround loop pattern; behavior can be confusing to maintainers
  - If execution context is altered unexpectedly, loop state can be difficult to debug
- **Sub-workflow reference:**  
  None.

---

## 2.4 Status Decision and Retry Handling

**Overview:**  
This block checks the current status of the ScraperCity job and branches based on whether it has succeeded. Non-complete jobs are retried after a delay.

**Nodes Involved:**  
- Check Job Status
- Is Job Complete?
- Wait 60 Seconds Before Retry

### Node: Check Job Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries ScraperCity for the current job state.
- **Configuration choices:**  
  - Method: `GET`
  - URL includes the current `runId`
  - HTTP Header Auth credential reused
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Poll Loop Controller`
  - Output: `Is Job Complete?`
- **Version-specific requirements:**  
  HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing/invalid `runId`
  - Auth error
  - API response format changes
  - Temporary ScraperCity outage or timeout
- **Sub-workflow reference:**  
  None.

### Node: Is Job Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the returned job status equals `SUCCEEDED`.
- **Configuration choices:**  
  - Strict string comparison
  - Condition: `$json.status === 'SUCCEEDED'`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Check Job Status`
  - True output: `Download Mobile Finder Results`
  - False output: `Wait 60 Seconds Before Retry`
- **Version-specific requirements:**  
  If node version `2.2`, condition format version `2`.
- **Edge cases or potential failure types:**  
  - Jobs in statuses like `FAILED`, `CANCELLED`, or other terminal error states are treated the same as “not complete” and retried forever
  - Case-sensitive comparison means `succeeded` or other variant would fail
- **Sub-workflow reference:**  
  None.

### Node: Wait 60 Seconds Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays the next polling attempt.
- **Configuration choices:**  
  Wait amount is set to `60` seconds.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Is Job Complete?` false branch
  - Output: `Poll Loop Controller`
- **Version-specific requirements:**  
  Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - Infinite retry loop is possible if the job never reaches `SUCCEEDED`
  - No max retry count or timeout guard is implemented
- **Sub-workflow reference:**  
  None.

---

## 2.5 Results Download and Parsing

**Overview:**  
After ScraperCity reports success, this block downloads the result file, attempts to parse it into structured records, and removes duplicates before downstream filtering.

**Nodes Involved:**  
- Download Mobile Finder Results
- Parse CSV Results
- Remove Duplicate Contacts

### Node: Download Mobile Finder Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed result payload from ScraperCity.
- **Configuration choices:**  
  - Method: `GET`
  - URL references `runId` from `Store Run ID`
  - Uses HTTP Header Auth
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input: `Is Job Complete?` true branch
  - Output: `Parse CSV Results`
- **Version-specific requirements:**  
  HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Download endpoint may differ by account/API version
  - If response is binary or not mapped into `.json.data`/`.json.body`, parsing can fail
  - Cross-node expression depends on `Store Run ID`
- **Sub-workflow reference:**  
  None.

### Node: Parse CSV Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts the download response into one n8n item per contact.
- **Configuration choices:**  
  The code:
  - tries `json.data` or `json.body` as raw CSV text
  - falls back to direct JSON handling if the response is already structured
  - splits by newline
  - uses first row as headers
  - maps each subsequent line to an object
  - attempts to detect a `phone` or `mobile` column
  - includes only records with phone data if such a column exists
  - otherwise passes all records through
  - emits a fallback info/error item if parsing fails or no phone rows are found
- **Key expressions or variables used:**  
  - `$input.first().json.data`
  - `$input.first().json.body`
  - `$input.first().json`
- **Input and output connections:**  
  - Input: `Download Mobile Finder Results`
  - Output: `Remove Duplicate Contacts`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - Naive CSV parsing with `split(',')` breaks on quoted commas within fields
  - CRLF line endings may leave stray `\r` characters in values
  - Binary download responses are not explicitly handled
  - Returns informational/error-like items that may still flow downstream
  - If no phone column exists, all records pass through
- **Sub-workflow reference:**  
  None.

### Node: Remove Duplicate Contacts
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Deduplicates records after parsing.
- **Configuration choices:**  
  Default options only; no specific fields are configured in the provided workflow.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Parse CSV Results`
  - Output: `Filter Records With Phone Numbers`
- **Version-specific requirements:**  
  Remove Duplicates node version `2`.
- **Edge cases or potential failure types:**  
  - Default duplicate logic may not match the intended contact identity
  - Records that differ slightly in formatting may survive as separate items
- **Sub-workflow reference:**  
  None.

---

## 2.6 Contact Filtering and Slack Message Preparation

**Overview:**  
This block limits the dataset to contacts that appear to contain phone fields and composes one aggregated Slack message containing all matching contacts.

**Nodes Involved:**  
- Filter Records With Phone Numbers
- Format Slack Message

### Node: Filter Records With Phone Numbers
- **Type and technical role:** `n8n-nodes-base.filter`  
  Retains only records where either `mobile_phone` or `phone` exists.
- **Configuration choices:**  
  OR logic with loose type validation:
  - `mobile_phone` exists
  - OR `phone` exists
- **Key expressions or variables used:**  
  - `={{ $json.mobile_phone }}`
  - `={{ $json.phone }}`
- **Input and output connections:**  
  - Input: `Remove Duplicate Contacts`
  - Output: `Format Slack Message`
- **Version-specific requirements:**  
  Filter node version `2.2`.
- **Edge cases or potential failure types:**  
  - Fields like `mobile`, `cell`, or other variants are not checked
  - “Exists” may pass empty-but-present values depending on actual payload shape
  - If no items pass, downstream may not execute because `alwaysOutputData` is false
- **Sub-workflow reference:**  
  None.

### Node: Format Slack Message
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates all contact items into a single Slack markdown message.
- **Configuration choices:**  
  The JavaScript code:
  - reads all incoming items
  - derives a contact name from `full_name`, `name`, or `first_name + last_name`
  - derives phone from `mobile_phone`, `phone`, or `mobile`
  - derives LinkedIn URL from `linkedin_url`, `linkedin`, or `input`
  - derives email from `email`
  - formats a numbered list
  - builds a summary header
  - includes `slackChannel` from `Store Run ID`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('Store Run ID').item.json.slackChannel`
- **Input and output connections:**  
  - Input: `Filter Records With Phone Numbers`
  - Output: `Alert Sales Team in Slack`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - The name expression is ambiguous because of JavaScript operator precedence; it may not behave exactly as intended for some records
  - Slack message can become very long if many contacts are returned
  - If no upstream items are received, this node will not run
- **Sub-workflow reference:**  
  None.

---

## 2.7 Slack Notification Delivery

**Overview:**  
This final block sends the assembled sales alert message to Slack using the configured bot credential and target channel.

**Nodes Involved:**  
- Alert Sales Team in Slack

### Node: Alert Sales Team in Slack
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a text message to a Slack channel.
- **Configuration choices:**  
  - Text: the generated Slack markdown message
  - Channel mode: by channel name
  - Markdown enabled via `mrkdwn: true`
- **Key expressions or variables used:**  
  - `={{ $json.slackMessage }}`
  - `={{ $json.slackChannel }}`
- **Input and output connections:**  
  - Input: `Format Slack Message`
  - Output: none
- **Version-specific requirements:**  
  Slack node version `2.3`.
- **Edge cases or potential failure types:**  
  - Slack credential missing or invalid
  - Bot lacks permission to post in the selected channel
  - Channel name resolution may fail if the bot is not a member
  - Large messages may be truncated or rejected depending on Slack limits
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow entry point |  | Configure Inputs | ## Manual trigger and config |
| Configure Inputs | Set | Defines LinkedIn URLs, emails, and Slack channel | When clicking 'Execute workflow' | Build Inputs Array | ## Manual trigger and config |
| Build Inputs Array | Code | Parses and combines manual inputs into ScraperCity request array | Configure Inputs | Submit Mobile Finder Job | ## Build and submit API job |
| Submit Mobile Finder Job | HTTP Request | Creates ScraperCity mobile-finder job | Build Inputs Array | Store Run ID | ## Build and submit API job |
| Store Run ID | Set | Stores ScraperCity run ID and Slack channel | Submit Mobile Finder Job | Wait 30 Seconds Before First Poll | ## Build and submit API job |
| Wait 30 Seconds Before First Poll | Wait | Delays first poll attempt | Store Run ID | Poll Loop Controller | ## Initial wait and poll loop |
| Poll Loop Controller | Split In Batches | Controls repeated status polling loop | Wait 30 Seconds Before First Poll; Wait 60 Seconds Before Retry | Check Job Status | ## Initial wait and poll loop |
| Check Job Status | HTTP Request | Fetches ScraperCity job status | Poll Loop Controller | Is Job Complete? | ## Job status check and retry |
| Is Job Complete? | If | Branches on `SUCCEEDED` status | Check Job Status | Download Mobile Finder Results; Wait 60 Seconds Before Retry | ## Job status check and retry |
| Wait 60 Seconds Before Retry | Wait | Delays before next polling cycle | Is Job Complete? | Poll Loop Controller | ## Job status check and retry |
| Download Mobile Finder Results | HTTP Request | Downloads completed result file | Is Job Complete? | Parse CSV Results | ## Download and parse results |
| Parse CSV Results | Code | Parses CSV-like response into contact items | Download Mobile Finder Results | Remove Duplicate Contacts | ## Download and parse results |
| Remove Duplicate Contacts | Remove Duplicates | Deduplicates parsed contacts | Parse CSV Results | Filter Records With Phone Numbers | ## Download and parse results |
| Filter Records With Phone Numbers | Filter | Keeps only records with phone fields | Remove Duplicate Contacts | Format Slack Message | ## Filter and format contacts |
| Format Slack Message | Code | Aggregates contacts into one Slack message | Filter Records With Phone Numbers | Alert Sales Team in Slack | ## Filter and format contacts |
| Alert Sales Team in Slack | Slack | Posts sales alert to Slack | Format Slack Message |  | ## Slack sales team alert |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Find mobile numbers from LinkedIn profiles and alert sales team in Slack`.

2. **Add a Manual Trigger node**.
   - Node type: `Manual Trigger`
   - Name it: `When clicking 'Execute workflow'`

3. **Add a Set node** after the trigger.
   - Node type: `Set`
   - Name it: `Configure Inputs`
   - Add these string fields:
     - `linkedinProfiles` → example: `linkedin.com/in/johndoe,linkedin.com/in/janedoe`
     - `workEmails` → example: `user@example.com,user@example.com`
     - `slackChannel` → example: `#sales-alerts`
   - Connect: `When clicking 'Execute workflow'` → `Configure Inputs`

4. **Add a Code node** to prepare the ScraperCity input array.
   - Node type: `Code`
   - Name it: `Build Inputs Array`
   - Use JavaScript that:
     - reads `linkedinProfiles` and `workEmails`
     - splits both strings on commas
     - trims whitespace
     - removes blanks
     - combines both arrays into one `inputs` array
     - returns `inputs` and `slackChannel`
   - Connect: `Configure Inputs` → `Build Inputs Array`

5. **Create ScraperCity credentials in n8n** before adding the request nodes.
   - Credential type: `HTTP Header Auth`
   - Suggested name: `ScraperCity API Key`
   - Configure the header expected by ScraperCity, typically something like an API key or bearer token header based on your account documentation.
   - Verify the required header name/value format in ScraperCity’s API docs.

6. **Add an HTTP Request node** to submit the mobile-finder job.
   - Node type: `HTTP Request`
   - Name it: `Submit Mobile Finder Job`
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/mobile-finder`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Header Auth`
   - Select credential: `ScraperCity API Key`
   - Body format:
     - Send body: enabled
     - Specify body: JSON
     - JSON body should send the `inputs` array from the previous node
   - Equivalent payload shape:
     - `{ "inputs": [...] }`
   - Connect: `Build Inputs Array` → `Submit Mobile Finder Job`

7. **Add a Set node** to preserve the run ID.
   - Node type: `Set`
   - Name it: `Store Run ID`
   - Add fields:
     - `runId` → expression from the API response field, expected as `{{$json.runId}}`
     - `slackChannel` → expression from `Build Inputs Array`, expected as `{{$('Build Inputs Array').item.json.slackChannel}}`
   - Connect: `Submit Mobile Finder Job` → `Store Run ID`

8. **Add a Wait node** for the first delay.
   - Node type: `Wait`
   - Name it: `Wait 30 Seconds Before First Poll`
   - Set amount to `30` seconds
   - Connect: `Store Run ID` → `Wait 30 Seconds Before First Poll`

9. **Add a Split In Batches node** to act as loop controller.
   - Node type: `Split In Batches`
   - Name it: `Poll Loop Controller`
   - Batch size: `1`
   - Reset: `false`
   - Connect: `Wait 30 Seconds Before First Poll` → `Poll Loop Controller`

10. **Add an HTTP Request node** to check job status.
    - Node type: `HTTP Request`
    - Name it: `Check Job Status`
    - Method: `GET`
    - URL expression:  
      `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
    - Authentication: `Generic Credential Type`
    - Generic Auth Type: `HTTP Header Auth`
    - Credential: `ScraperCity API Key`
    - Connect: `Poll Loop Controller` → `Check Job Status`

11. **Add an If node** to evaluate completion.
    - Node type: `If`
    - Name it: `Is Job Complete?`
    - Create a condition:
      - Left value: `{{$json.status}}`
      - Operator: `equals`
      - Right value: `SUCCEEDED`
    - Keep comparison case-sensitive/strict as in the original workflow
    - Connect: `Check Job Status` → `Is Job Complete?`

12. **Add a second Wait node** for retries.
    - Node type: `Wait`
    - Name it: `Wait 60 Seconds Before Retry`
    - Set amount to `60` seconds
    - Connect the **false** output of `Is Job Complete?` to this node

13. **Loop the retry back** into the polling controller.
    - Connect: `Wait 60 Seconds Before Retry` → `Poll Loop Controller`

14. **Add an HTTP Request node** to download results.
    - Node type: `HTTP Request`
    - Name it: `Download Mobile Finder Results`
    - Method: `GET`
    - URL expression:  
      `https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
    - Authentication: `Generic Credential Type`
    - Generic Auth Type: `HTTP Header Auth`
    - Credential: `ScraperCity API Key`
    - Connect the **true** output of `Is Job Complete?` to this node

15. **Add a Code node** to parse the downloaded results.
    - Node type: `Code`
    - Name it: `Parse CSV Results`
    - Add JavaScript that:
      - reads raw data from `json.data` or `json.body`
      - handles fallback if the response is already structured JSON
      - splits CSV text into lines
      - treats the first line as headers
      - maps each remaining line to an object
      - optionally detects `phone`/`mobile` columns and keeps rows with phone values
      - emits one item per contact
    - Connect: `Download Mobile Finder Results` → `Parse CSV Results`

16. **Add a Remove Duplicates node**.
    - Node type: `Remove Duplicates`
    - Name it: `Remove Duplicate Contacts`
    - Leave default configuration unless you want to define a specific dedupe key
    - Connect: `Parse CSV Results` → `Remove Duplicate Contacts`

17. **Add a Filter node** for records with phone numbers.
    - Node type: `Filter`
    - Name it: `Filter Records With Phone Numbers`
    - Set OR conditions:
      - `{{$json.mobile_phone}}` exists
      - OR `{{$json.phone}}` exists
    - Use loose type validation if you want behavior close to the original
    - Connect: `Remove Duplicate Contacts` → `Filter Records With Phone Numbers`

18. **Add a Code node** to build the Slack message.
    - Node type: `Code`
    - Name it: `Format Slack Message`
    - Add JavaScript that:
      - reads all incoming contact items
      - builds one line per contact with name, phone, email, and LinkedIn profile
      - prefixes the list with a summary like `Mobile Finder Results`
      - returns:
        - `slackMessage`
        - `contactCount`
        - `slackChannel` from `Store Run ID`
    - Connect: `Filter Records With Phone Numbers` → `Format Slack Message`

19. **Create Slack credentials in n8n**.
    - Credential type: `Slack API`
    - Suggested name: `Slack API Credential`
    - Use a Slack app/bot token with permission to post messages
    - Required scope noted in the workflow comments: `chat:write`
    - Ensure the bot is installed to the workspace and added to the target channel if needed

20. **Add a Slack node** to send the alert.
    - Node type: `Slack`
    - Name it: `Alert Sales Team in Slack`
    - Operation: send message
    - Text: `{{$json.slackMessage}}`
    - Channel selection mode: by name
    - Channel value: `{{$json.slackChannel}}`
    - Enable markdown / mrkdwn
    - Credential: `Slack API Credential`
    - Connect: `Format Slack Message` → `Alert Sales Team in Slack`

21. **Test the workflow manually**.
    - Open `Configure Inputs`
    - Insert real LinkedIn profile URLs, work emails, and a valid Slack channel
    - Click `Execute workflow`
    - Watch the execution log through:
      - job submission
      - waiting
      - polling loop
      - result download
      - Slack delivery

22. **Validate endpoint and response assumptions**.
    - Confirm ScraperCity returns `runId` on submission
    - Confirm status endpoint returns `status`
    - Confirm download endpoint returns CSV text accessible by the parser
    - Adjust endpoints if your ScraperCity account uses another API version or path

23. **Recommended hardening improvements if rebuilding for production**:
    - Add validation for empty input arrays
    - Add a max retry counter or timeout for polling
    - Handle terminal failure statuses like `FAILED`
    - Use a robust CSV parser instead of `split(',')`
    - Add a fallback Slack message when no phone numbers are found
    - Deduplicate on explicit fields such as email or LinkedIn URL

**Sub-workflow setup:**  
There are no sub-workflows or Execute Workflow nodes in this workflow. No child workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Find mobile numbers from LinkedIn profiles and alert sales team in Slack | Workflow title / purpose |
| The workflow is triggered manually and configured with a list of LinkedIn profile URLs, work emails, and a target Slack channel. | General workflow behavior |
| Inputs are combined into a structured array and submitted as a mobile-finder job to the ScraperCity API. | ScraperCity job submission |
| The workflow waits 30 seconds, then enters a polling loop that checks job status every 60 seconds until the job is complete. | Polling behavior |
| Once complete, the results CSV is downloaded, parsed into individual contact records, and deduplicated. | Result handling |
| Records without phone numbers are filtered out, and each remaining contact is formatted into a Slack message block. | Filtering and formatting |
| The sales team is alerted in the configured Slack channel with the discovered mobile numbers. | Slack notification |
| Obtain a ScraperCity API key and add it as an n8n credential for the HTTP Request nodes. | Credential prerequisite |
| Configure a Slack bot/app with `chat:write` permissions and add the credential to n8n. | Slack prerequisite |
| Open the 'Configure Inputs' node and populate `linkedinProfiles`, `workEmails`, and `slackChannel` with your target data. | Setup instruction |
| Verify the ScraperCity endpoint URLs in 'Submit Mobile Finder Job', 'Check Job Status', and 'Download Mobile Finder Results' match your account's API version. | Setup instruction |
| Run the workflow manually using 'Execute workflow' and monitor execution logs to confirm polling and results. | Execution instruction |
| Adjust the wait durations in 'Wait 30 Seconds Before First Poll' and 'Wait 60 Seconds Before Retry' based on typical ScraperCity job completion times. | Customization note |
| The 'Format Slack Message' code node can be edited to change message layout, add contact fields, or include links back to LinkedIn profiles. | Customization note |