Retweet Cleanup with Scheduling for X/Twitter

https://n8nworkflows.xyz/workflows/retweet-cleanup-with-scheduling-for-x-twitter-9766


# Retweet Cleanup with Scheduling for X/Twitter

### 1. Workflow Overview

This workflow automates the cleanup of retweets on specified X (formerly Twitter) accounts by unretweeting them on a scheduled basis. It is designed for social media managers, creators, and brand accounts that want to maintain a clean and relevant feed while leveraging retweets temporarily for campaigns or events. The workflow fetches recent tweets, filters retweets, and deletes (unretweets) them in rate-limit-friendly batches with configurable delays.

**Logical Blocks:**

- **1.1 Input Reception & Scheduling:** Triggers the workflow daily at a configured time.
- **1.2 Configuration Setup:** Centralizes user-defined variables for easy parameter adjustments.
- **1.3 User Identification:** Resolves a Twitter handle to a user ID via the X API.
- **1.4 Tweet Retrieval:** Fetches recent tweets for the user, including metadata to detect retweets.
- **1.5 Tweet Processing & Filtering:** Splits tweet arrays into individual items and filters only retweets.
- **1.6 Batch Processing & API Rate Management:** Processes retweets in batches, unretweets them, and spaces calls with delays.
- **1.7 Optional Notifications & Error Handling:** Sends notifications (e.g., Slack) after each unretweet and handles runtime errors with dead-letter queue (DLQ) formatting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

- **Overview:**  
  Initiates the workflow automatically every day at 09:00 server time. Users can adjust the schedule cadence.

- **Nodes Involved:**  
  - TRIGGER | Schedule (09:00 server time)

- **Node Details:**

  - **TRIGGER | Schedule (09:00 server time):**
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution at a fixed daily time.
    - Configuration: Set to trigger once daily at 09:00 server timezone.
    - Input/Output: No input; outputs a single empty item to start the flow.
    - Edge Cases: Misconfigured cron or interval can prevent execution.
    - Notes: No direct failure modes; manual trigger recommended for testing.
    - Sticky Note Reference: "Sticky: TRIGGER details"

#### 2.2 Configuration Setup

- **Overview:**  
  Centralizes user-editable variables such as target Twitter handle, max tweets to fetch, and delay between API calls for easy modification without changing node logic.

- **Nodes Involved:**  
  - CONFIG | User Variables (Set Fields)

- **Node Details:**

  - **CONFIG | User Variables (Set Fields):**
    - Type: Set  
    - Role: Stores configuration parameters as JSON fields.
    - Configuration:  
      - `target_username`: Twitter handle (string, no '@')  
      - `max_results`: Number of tweets to fetch (default 50, max 100)  
      - `batch_delay_minutes`: Delay between API calls in minutes (default 1)
    - Input/Output: No input; outputs JSON with these fields.
    - Edge Cases: Incorrect or missing values can affect downstream API queries.
    - Sticky Note Reference: "Sticky: CONFIG details"

#### 2.3 User Identification

- **Overview:**  
  Resolves the Twitter username into a unique user ID via the X API to use in subsequent tweet fetch requests.

- **Nodes Involved:**  
  - FETCH | Get user ID by username (X API)

- **Node Details:**

  - **FETCH | Get user ID by username (X API):**
    - Type: HTTP Request  
    - Role: Calls `GET /2/users/by/username/{username}` to resolve user ID.
    - Configuration:  
      - URL dynamically constructed from `target_username` in CONFIG node.  
      - Uses OAuth2 credentials (no tokens embedded).  
      - Headers include User-Agent "n8n-workflow".
    - Input: Receives configuration JSON.  
    - Output: JSON object with user ID under `data.id`.
    - Edge Cases:  
      - 401/403 errors if OAuth2 credentials are invalid or expired.  
      - 404 if username not found or suspended.
    - Sticky Note Reference: "Sticky: GET USER details"

#### 2.4 Tweet Retrieval

- **Overview:**  
  Fetches the latest tweets posted by the user, including metadata fields required for retweet detection.

- **Nodes Involved:**  
  - FETCH | Latest tweets (X API)

- **Node Details:**

  - **FETCH | Latest tweets (X API):**
    - Type: HTTP Request  
    - Role: Calls `GET /2/users/{id}/tweets` to retrieve recent tweets.
    - Configuration:  
      - URL built dynamically from user ID from previous node.  
      - Query parameters:  
        - `max_results` from CONFIG node (up to 100).  
        - `tweet.fields=created_at,referenced_tweets` to get retweet info.  
      - Uses OAuth2 credentials.
    - Input: User ID JSON.  
    - Output: JSON array of tweets under `data`.
    - Edge Cases:  
      - Empty `data` array if no tweets.  
      - Pagination not implemented; only latest batch fetched.
    - Sticky Note Reference: "Sticky: FETCH TWEETS details"

#### 2.5 Tweet Processing & Filtering

- **Overview:**  
  Transforms the array of tweets into individual tweet items and filters them to retain only retweets for further processing.

- **Nodes Involved:**  
  - SPLIT | One item per tweet  
  - IF | Is retweet?

- **Node Details:**

  - **SPLIT | One item per tweet:**
    - Type: Split Out  
    - Role: Converts the array of tweets (`data`) into separate items for individual processing.
    - Configuration:  
      - Splits on field `data`.
    - Input: JSON with array of tweets.  
    - Output: One item per tweet.
    - Edge Cases: If API payload changes field names, this must be updated.
    - Sticky Note Reference: "Sticky: SPLIT details"

  - **IF | Is retweet?:**
    - Type: If  
    - Role: Checks if the tweet is a retweet by inspecting the first referenced tweet type.
    - Configuration:  
      - Condition: `$json.referenced_tweets ? $json.referenced_tweets[0].type : ''` equals `"retweeted"`.
    - Input: Single tweet JSON.  
    - Output: True branch for retweets; false branch drops non-retweets.
    - Edge Cases: Missing `referenced_tweets` or empty array leads to false branch.  
      Can be customized to filter other tweet types (replies, quotes).
    - Sticky Note Reference: "Sticky: IF RT details"

#### 2.6 Batch Processing & API Rate Management

- **Overview:**  
  Processes filtered retweets in small batches to respect API rate limits, deletes (unretweets) them, and spaces calls with configurable delays.

- **Nodes Involved:**  
  - LOOP | Split in batches (rate-limit friendly)  
  - SEND | Unretweet (delete retweet)  
  - WAIT | Delay between API calls  
  - NOTIFY | Slack (optional)

- **Node Details:**

  - **LOOP | Split in batches (rate-limit friendly):**
    - Type: Split In Batches  
    - Role: Divides the stream of retweet items into manageable batches for sequential processing.
    - Configuration: Default batch size (configurable if needed).
    - Input: Retweet tweet items.  
    - Output: Batches of tweets to delete.
    - Edge Cases: API failures can be routed to error handling.
    - Sticky Note Reference: "Sticky: LOOP details"

  - **SEND | Unretweet (delete retweet):**
    - Type: Twitter  
    - Role: Deletes the retweet by tweet ID, effectively unretweeting it.
    - Configuration:  
      - Operation: `delete`  
      - Tweet ID to delete: dynamically set as `$json.id` of the retweet item.  
      - Uses OAuth2 Twitter credentials.
    - Input: Individual tweet JSON from batch.  
    - Output: Confirmation or error data.
    - Edge Cases:  
      - 404 if tweet ID is not a retweet or already deleted.  
      - 403 if credentials lack write/delete scope.
    - Sticky Note Reference: "Sticky: UNRETWEET details"

  - **WAIT | Delay between API calls:**
    - Type: Wait  
    - Role: Inserts a delay between API calls to avoid rate limit errors (e.g., 429).
    - Configuration:  
      - Delay unit: minutes  
      - Delay amount: `batch_delay_minutes` from CONFIG node (recommended ≥ 1).
    - Input: Output of unretweet operation.  
    - Output: Passes item downstream after delay.
    - Edge Cases: Longer delays may be needed during heavy cleanup.
    - Sticky Note Reference: "Sticky: WAIT details"

  - **NOTIFY | Slack (optional):**
    - Type: Slack  
    - Role: Sends informational Slack messages after each unretweet action.
    - Configuration:  
      - Channel set to a configured Slack channel (e.g., `#your-channel`).  
      - Message text derived from `$json.message` (assumed to be formatted upstream).  
      - Requires Slack credentials.
    - Input: Output from WAIT node.  
    - Output: Slack API response.
    - Edge Cases: Slack API failures do not break main flow.
    - Sticky Note Reference: "Sticky: DLQ & Notify"

#### 2.7 Optional Notifications & Error Handling

- **Overview:**  
  Captures workflow runtime errors, formats error data for dead-letter queue (DLQ) destinations, and optionally connects to external systems for alerting and auditing.

- **Nodes Involved:**  
  - ON ERROR | Capture  
  - DLQ | Format error (Set)

- **Node Details:**

  - **ON ERROR | Capture:**
    - Type: Error Trigger  
    - Role: Captures any runtime errors in the workflow execution.
    - Configuration: None (built-in).
    - Output: Passes error data on workflow failure.
    - Edge Cases: Must be connected to DLQ or notification nodes to act on errors.
    - Sticky Note Reference: "Sticky: DLQ & Notify"

  - **DLQ | Format error (Set):**
    - Type: Set  
    - Role: Formats error details for logging or external storage.
    - Configuration:  
      - Sets fields:  
        - `failed_node` = name of node where error occurred  
        - `error_message` = error message text  
        - `timestamp` = current ISO timestamp
    - Input: Error JSON from ON ERROR node.  
    - Output: Formatted error record.
    - Edge Cases: Can be extended to send errors to Google Sheets, Email, Slack.
    - Sticky Note Reference: "Sticky: DLQ & Notify"

---

### 3. Summary Table

| Node Name                           | Node Type                 | Functional Role                                | Input Node(s)                          | Output Node(s)                           | Sticky Note                                                |
|-----------------------------------|---------------------------|------------------------------------------------|---------------------------------------|-----------------------------------------|------------------------------------------------------------|
| TRIGGER | Schedule (09:00 server time) | Schedule Trigger           | None                                          | CONFIG | User Variables (Set Fields)    | Runs daily at 09:00 (server timezone). Adjust cadence as needed. Sticky: TRIGGER details |
| CONFIG | User Variables (Set Fields) | Set                       | TRIGGER | Schedule (09:00 server time)          | FETCH | Get user ID by username (X API)   | Centralizes user-editable variables for easy setup. Sticky: CONFIG details                   |
| FETCH | Get user ID by username (X API) | HTTP Request              | CONFIG | User Variables (Set Fields)                  | FETCH | Latest tweets (X API)              | Resolves @handle → user.id via X API. Uses OAuth2 Credentials. Sticky: GET USER details      |
| FETCH | Latest tweets (X API)           | HTTP Request              | FETCH | Get user ID by username (X API)              | SPLIT | One item per tweet               | Fetches recent tweets for the user. Includes retweet detection fields. Sticky: FETCH TWEETS details |
| SPLIT | One item per tweet              | Split Out                 | FETCH | Latest tweets (X API)                         | IF | Is retweet?                     | Converts tweets array into individual tweet items. Sticky: SPLIT details                    |
| IF | Is retweet?                     | If                        | SPLIT | One item per tweet                            | LOOP | Split in batches (rate-limit friendly) | Filters only retweets based on referenced_tweets type. Sticky: IF RT details                |
| LOOP | Split in batches (rate-limit friendly) | Split In Batches           | IF | Is retweet?                                  | SEND | Unretweet (delete retweet)       | Processes items in small batches to respect API limits. Sticky: LOOP details                |
| SEND | Unretweet (delete retweet)      | Twitter                   | LOOP | Split in batches (rate-limit friendly)        | WAIT | Delay between API calls          | Deletes your retweet tweet by ID; original tweet remains. Sticky: UNRETWEET details         |
| WAIT | Delay between API calls          | Wait                      | SEND | Unretweet (delete retweet)                    | NOTIFY | Slack (optional)                | Spaces API calls using batch_delay_minutes from CONFIG. Sticky: WAIT details                |
| NOTIFY | Slack (optional)                | Slack                     | WAIT | Delay between API calls                        | LOOP | Split in batches (rate-limit friendly) | Optional Slack notification after each unretweet. Sticky: DLQ & Notify                     |
| ON ERROR | Capture                      | Error Trigger             | None                                          | DLQ | Format error (Set)               | Catches workflow runtime errors. Sticky: DLQ & Notify                                      |
| DLQ | Format error (Set)              | Set                       | ON ERROR | Capture                                    | None                                    | Formats error record for dead-letter queue or notifications. Sticky: DLQ & Notify          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Node Type: Schedule Trigger  
   - Name: TRIGGER | Schedule (09:00 server time)  
   - Configuration: Set to trigger daily at 09:00 server time.

2. **Create Configuration Node:**
   - Node Type: Set  
   - Name: CONFIG | User Variables (Set Fields)  
   - Parameters (Set fields):  
     - `target_username` (string): your X handle without '@'  
     - `max_results` (number): default 50 (max 100)  
     - `batch_delay_minutes` (number): default 1 (minimum 1 recommended)  
   - Connect TRIGGER node output to this node.

3. **Create User ID Fetch Node:**
   - Node Type: HTTP Request  
   - Name: FETCH | Get user ID by username (X API)  
   - Authentication: OAuth2 (assign your X developer app credentials)  
   - HTTP Method: GET  
   - URL: `https://api.twitter.com/2/users/by/username/{{ $json.target_username }}`  
   - Headers:  
     - User-Agent: n8n-workflow  
   - Connect CONFIG node output to this node.

4. **Create Latest Tweets Fetch Node:**
   - Node Type: HTTP Request  
   - Name: FETCH | Latest tweets (X API)  
   - Authentication: OAuth2 (same credentials)  
   - HTTP Method: GET  
   - URL: `https://api.twitter.com/2/users/{{ $json.data.id }}/tweets`  
   - Query Parameters:  
     - max_results: `{{ $json.max_results }}`  
     - tweet.fields: `created_at,referenced_tweets`  
   - Connect User ID Fetch node output to this node.

5. **Create Split Tweets Node:**
   - Node Type: Split Out  
   - Name: SPLIT | One item per tweet  
   - Field to split out: `data` (array of tweets)  
   - Connect Tweets Fetch node output to this node.

6. **Create Retweet Filter Node:**
   - Node Type: If  
   - Name: IF | Is retweet?  
   - Condition: String equals  
     - Left: `{{ $json.referenced_tweets ? $json.referenced_tweets[0].type : '' }}`  
     - Right: `retweeted`  
   - Connect Split node output to this node.

7. **Create Batch Loop Node:**
   - Node Type: Split In Batches  
   - Name: LOOP | Split in batches (rate-limit friendly)  
   - Connect IF node’s True output (retweets) to this node.

8. **Create Unretweet Node:**
   - Node Type: Twitter  
   - Name: SEND | Unretweet (delete retweet)  
   - Operation: Delete Tweet  
   - Tweet Delete ID: `{{ $json.id }}`  
   - Credentials: OAuth2 with write/delete permissions  
   - Connect Batch Loop node output to this node.

9. **Create Wait Node:**
   - Node Type: Wait  
   - Name: WAIT | Delay between API calls  
   - Unit: Minutes  
   - Amount: `{{ $json.batch_delay_minutes }}` (from CONFIG node)  
   - Connect Unretweet node output to this node.

10. **Create Optional Slack Notification Node:**
    - Node Type: Slack  
    - Name: NOTIFY | Slack (optional)  
    - Channel: Your Slack channel (e.g., `#your-channel`)  
    - Text: `{{ $json.message }}` (you will need to prepare this message upstream if desired)  
    - Connect Wait node output to this node.  
    - Connect Slack node output back to Batch Loop node to continue processing remaining items.

11. **Create Error Handling Nodes:**

    - **Error Trigger:**
      - Node Type: Error Trigger  
      - Name: ON ERROR | Capture

    - **Format Error Node:**
      - Node Type: Set  
      - Name: DLQ | Format error (Set)  
      - Fields:  
        - `failed_node`: `{{ $json.execution.error.node.name }}`  
        - `error_message`: `{{ $json.execution.error.message }}`  
        - `timestamp`: `{{ new Date().toISOString() }}`  
      - Connect ON ERROR node output to this node.

    - Optionally connect DLQ output to external destinations like Google Sheets, Email, or Slack for error logging and alerting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Who’s it for: Social media managers, creators, brand accounts needing automated cleanup of retweets after campaigns or events to keep profiles tidy and relevant.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Description sticky note in workflow                                                                |
| How to customize: Increase `max_results` up to 100, change schedule timing, modify IF condition to filter replies or quotes, add logging or notification nodes to Slack or Email.                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky notes "Rules & customization" and "DLQ & Notify"                                            |
| Troubleshooting common errors: 401/403 due to invalid OAuth2 scopes or expired tokens; 404 due to invalid tweet ID or missing delete permissions; 429 rate limits require adding or increasing wait delays and reducing batch sizes.                                                                                                                                                                                                                                                                                                                                                                                        | Sticky note "Troubleshooting"                                                                       |
| Security: No hardcoded API keys or tokens; all credentials managed via OAuth2 credential manager in n8n. Keep batch delay ≥ 1 minute to avoid rate limiting.                                                                                                                                                                                                                                                                                                                                                                                                                                                               | General best practice from workflow description                                                    |
| API endpoints used:  
   - GET `/2/users/by/username/{username}` to resolve user ID  
   - GET `/2/users/{id}/tweets?tweet.fields=created_at,referenced_tweets` to fetch tweets with retweet info  
   Original tweet is never deleted by this workflow, only your retweets.                                                                                                                                                                                                                                                                                                                                                                                   | Sticky note "API endpoints"                                                                        |
| Data flow summary: TRIGGER → CONFIG → GET USER → FETCH TWEETS → SPLIT → IF RT → LOOP → UNRETWEET → WAIT → NOTIFY → LOOP (continue)  
   Optional error path: ON ERROR → DLQ → external systems (Sheets, Email, Slack) for audit and alerting                                                                                                                                                                                                                                                                                                                                                                                   | Sticky note "Data flow map"                                                                        |
| Quick setup instructions:  
    1. Edit CONFIG node to set your `target_username`, `max_results`, `batch_delay_minutes`.  
    2. Assign OAuth2 credentials for X (Twitter) in HTTP request and Twitter nodes.  
    3. Run once manually to test, then enable schedule for automation.                                                                                                                                                                                                                                                                                                                                                                   | Sticky note "Quick setup"                                                                          |

---

**Disclaimer:** This documentation is derived exclusively from an n8n workflow automation designed for Twitter retweet cleanup. The workflow complies with current content policies and handles only legal, public data.