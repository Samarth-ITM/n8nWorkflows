Scheduled Auto-replies to Targeted Tweets Using X (Twitter) API

https://n8nworkflows.xyz/workflows/scheduled-auto-replies-to-targeted-tweets-using-x--twitter--api-10781


# Scheduled Auto-replies to Targeted Tweets Using X (Twitter) API

### 1. Workflow Overview

This workflow automates scheduled auto-replies to targeted tweets on X (formerly Twitter) using the Twitter API v2. It runs on a fixed interval (every 30 minutes), searches for recent tweets from specified users, prepares a predefined reply, and posts that reply while managing API rate limits and preventing duplicate responses.

The workflow is logically divided into three main blocks:

- **1.1 Scheduled Tweet Search**: Periodically triggers and searches recent tweets from a list of target Twitter users using the Twitter API v2.
- **1.2 Reply Preparation and Sending**: Processes the retrieved tweets to extract necessary information, prepares a reply message, and sends replies via the Twitter API.
- **1.3 Error Handling and Status Reporting**: Handles different error scenarios such as API rate limiting, no tweets found, duplicate replies, and other possible API errors, providing detailed status messages accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Tweet Search

- **Overview:**  
  This block triggers the workflow on a schedule (every 30 minutes) and performs a search query on Twitter to find recent tweets from specific user accounts. It aggregates and filters the latest tweets per author for further processing.

- **Nodes Involved:**  
  - Schedule Trigger (15分ごと)  
  - Search Tweets (HTTP)  
  - Get Latest Tweet Per Account  

- **Node Details:**

  1. **Schedule Trigger (15分ごと)**  
     - *Type & Role:* Schedule Trigger node, initiates workflow execution every 30 minutes.  
     - *Configuration:* Interval set to 30 minutes (field: minutesInterval=30).  
     - *Input/Output:* No input; outputs a trigger event to "Search Tweets (HTTP)".  
     - *Edge cases:* None, but if the n8n instance is down, triggers may not run on time.

  2. **Search Tweets (HTTP)**  
     - *Type & Role:* HTTP Request node to call Twitter API v2's recent search endpoint.  
     - *Configuration:*  
       - URL: `https://api.twitter.com/2/tweets/search/recent`  
       - HTTP Method: POST (implicit through n8n HTTP Request)  
       - Query Parameters:  
         - `query`: searches tweets from specific users (e.g., from:badassceo OR from:bozu_108 OR from:hirox246 OR from:takaichi_sanae -is:retweet)  
         - `tweet.fields`: includes creation date and author ID  
         - `expansions`: expands author_id to user objects  
         - `user.fields`: includes username  
         - `max_results`: limits to 50 tweets per request  
       - Authentication: Twitter OAuth2 credentials required.  
     - *Input:* Trigger from schedule node.  
     - *Output:* Raw Twitter API response JSON forwarded to "Get Latest Tweet Per Account".  
     - *Edge cases:* API rate limits, authentication failures, network timeouts, malformed query parameters.

  3. **Get Latest Tweet Per Account**  
     - *Type & Role:* Code node that merges multiple API responses, filters tweets by author, and returns only the latest tweet per author.  
     - *Configuration:*  
       - Custom JavaScript code merges arrays from multiple responses (`items`), extracts tweets and users, checks for errors (rate limit, auth, bad request), and outputs the latest tweet per author with user info.  
       - Returns status "no_tweets" if no tweets found or "error" with details if API error detected.  
     - *Input:* Output from Search Tweets (HTTP).  
     - *Output:* Filtered tweets or error status passed to next conditional node "Is Success?".  
     - *Edge cases:* Possible errors from Twitter API handled here; robust error classification implemented.

---

#### 1.2 Reply Preparation and Sending

- **Overview:**  
  Takes the filtered tweets, prepares a fixed reply message including tweet and user info, sends the reply via Twitter API, and handles duplicate content errors.

- **Nodes Involved:**  
  - Is Success?  
  - Prepare Reply  
  - Send Reply (HTTP)  
  - Is Duplicate?  
  - Reply Success  
  - Already Replied (Skip)  
  - Reply Error  

- **Node Details:**

  1. **Is Success?**  
     - *Type & Role:* If node checking if the previous step produced a "success" status.  
     - *Configuration:* Checks if `$json.status` equals `"success"`.  
     - *Input:* From "Get Latest Tweet Per Account".  
     - *Outputs:*  
       - True branch: to "Prepare Reply" (process tweets for reply).  
       - False branch: to "Is Rate Limit?" for error handling.  
     - *Edge cases:* Status value missing or unexpected.

  2. **Prepare Reply**  
     - *Type & Role:* Code node that constructs the reply message and extracts user info per tweet.  
     - *Configuration:*  
       - JavaScript code extracts tweet text, tweet ID, author ID, and username from included user data.  
       - Defines a fixed reply message:  
         `弊社の新サービスをご覧ください！ https://x.com/linepi_pi/status/1988478476761133063`  
       - Outputs JSON with fields: `replyText`, `replyToTweetId`, `extractedUsername`, `originalTweet`.  
     - *Input:* True branch of "Is Success?".  
     - *Output:* To "Send Reply (HTTP)".  
     - *Edge cases:* Missing user info or unexpected data structure.

  3. **Send Reply (HTTP)**  
     - *Type & Role:* HTTP Request node that posts the reply tweet via Twitter API v2.  
     - *Configuration:*  
       - URL: `https://api.twitter.com/2/tweets`  
       - Method: POST  
       - Body (JSON):  
         `{ "text": $json.replyText, "reply": { "in_reply_to_tweet_id": $json.replyToTweetId } }`  
       - Auth: Twitter OAuth2 credentials.  
       - Continue On Fail enabled to catch errors without stopping workflow.  
     - *Input:* Output from "Prepare Reply".  
     - *Output:* To "Is Duplicate?" for checking possible duplicate content errors.  
     - *Edge cases:* API errors such as duplicate content or rate limits.

  4. **Is Duplicate?**  
     - *Type & Role:* If node checking if the reply failed due to duplicate content error.  
     - *Configuration:* Checks if the error message contains `"duplicate content"`.  
     - *Input:* Output from "Send Reply (HTTP)".  
     - *Outputs:*  
       - True branch: to "Already Replied (Skip)".  
       - False branch: to "Reply Success".  
     - *Edge cases:* Error message format variations.

  5. **Reply Success**  
     - *Type & Role:* Set node to log successful reply details.  
     - *Configuration:* Assigns status `"✅ リプライ送信成功"` with relevant info such as tweet ID replied to, timestamp, reply text, username, and API response.  
     - *Input:* False branch from "Is Duplicate?".  
     - *Output:* Terminal or further logging (not connected).  
     - *Edge cases:* None.

  6. **Already Replied (Skip)**  
     - *Type & Role:* Set node to log skipped duplicate replies.  
     - *Configuration:* Assigns status `"✅ 既にリプライ済み（スキップ）"` with details explaining no duplicate reply sent, includes tweet ID and username.  
     - *Input:* True branch from "Is Duplicate?".  
     - *Output:* Terminal or logging.  
     - *Edge cases:* None.

  7. **Reply Error**  
     - *Type & Role:* Set node to log errors encountered during reply sending.  
     - *Configuration:* Assigns status `"❌ リプライ送信エラー"` and stores error message, tweet ID, timestamp, and full API response.  
     - *Input:* This node is defined but not directly connected in the current workflow JSON — presumably intended for future error handling or manual trigger.  
     - *Edge cases:* Capture unexpected reply failures.

---

#### 1.3 Error Handling and Status Reporting

- **Overview:**  
  This block manages different error cases such as API rate limits, no tweets found, and other errors, setting appropriate status messages and timestamps.

- **Nodes Involved:**  
  - Is Rate Limit?  
  - Rate Limit Error  
  - Is No Tweets?  
  - No Tweet Found  
  - Other Error  

- **Node Details:**

  1. **Is Rate Limit?**  
     - *Type & Role:* If node checking if error type equals `"rate_limit"`.  
     - *Configuration:* Checks if `$json.errorType === "rate_limit"`.  
     - *Input:* False branch from "Is Success?" (i.e., error condition).  
     - *Outputs:*  
       - True branch: to "Rate Limit Error".  
       - False branch: to "Is No Tweets?".  
     - *Edge cases:* Error type field missing or incorrect.

  2. **Rate Limit Error**  
     - *Type & Role:* Set node to log API rate limiting status.  
     - *Configuration:* Sets status `"⚠️ API制限エラー"` and message explaining the 15-minute automatic recovery window, includes error detail and timestamp.  
     - *Input:* True branch from "Is Rate Limit?".  
     - *Output:* Terminal or logging.  
     - *Edge cases:* None.

  3. **Is No Tweets?**  
     - *Type & Role:* If node checking if status equals `"no_tweets"`.  
     - *Configuration:* Checks `$json.status === "no_tweets"`.  
     - *Input:* False branch from "Is Rate Limit?".  
     - *Outputs:*  
       - True branch: to "No Tweet Found".  
       - False branch: to "Other Error".  
     - *Edge cases:* Unexpected status values.

  4. **No Tweet Found**  
     - *Type & Role:* Set node to log that no monitored user tweets were found.  
     - *Configuration:* Sets status `"📭 ツイートなし"`, reason message, and timestamp.  
     - *Input:* True branch from "Is No Tweets?".  
     - *Output:* Terminal or logging.  
     - *Edge cases:* None.

  5. **Other Error**  
     - *Type & Role:* Set node for logging other unknown errors.  
     - *Configuration:* Sets status `"❌ その他のエラー"`, error type and message fields, and timestamp.  
     - *Input:* False branch from "Is No Tweets?".  
     - *Output:* Terminal or logging.  
     - *Edge cases:* Handles any other API or processing errors not categorized above.

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                       | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                              |
|-----------------------------|----------------------|------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger (15分ごと)  | Schedule Trigger     | Triggers workflow every 30 minutes | -                           | Search Tweets (HTTP)        | ## 1. Search for Tweets This section runs on a schedule and fetches new tweets based on your query.     |
| Search Tweets (HTTP)         | HTTP Request         | Calls Twitter API for recent tweets| Schedule Trigger             | Get Latest Tweet Per Account|                                                                                                        |
| Get Latest Tweet Per Account | Code                 | Filters latest tweet per author     | Search Tweets (HTTP)         | Is Success?                 |                                                                                                        |
| Is Success?                  | If                   | Checks if tweets found successfully | Get Latest Tweet Per Account | Prepare Reply / Is Rate Limit? |                                                                                                        |
| Prepare Reply               | Code                 | Prepares reply message and metadata | Is Success? (true)           | Send Reply (HTTP)           | ## 2. Process & Send Reply This section prepares and sends your defined reply to the new tweets.        |
| Send Reply (HTTP)            | HTTP Request         | Sends reply tweet via Twitter API   | Prepare Reply                | Is Duplicate?               |                                                                                                        |
| Is Duplicate?                | If                   | Detects duplicate reply errors      | Send Reply (HTTP)            | Already Replied (Skip) / Reply Success |                                                                                                        |
| Reply Success                | Set                  | Logs successful reply                | Is Duplicate? (false)        | -                          |                                                                                                        |
| Already Replied (Skip)       | Set                  | Logs skipped duplicate reply        | Is Duplicate? (true)         | -                          |                                                                                                        |
| Reply Error                 | Set                  | Logs reply sending errors            | -                           | -                          |                                                                                                        |
| Is Rate Limit?               | If                   | Detects API rate limit errors        | Is Success? (false)          | Rate Limit Error / Is No Tweets? |                                                                                                        |
| Rate Limit Error             | Set                  | Logs rate limit error                | Is Rate Limit? (true)        | -                          | ## 3. Error Handling This section catches common errors like API rate limits or empty search results.    |
| Is No Tweets?                | If                   | Checks if no tweets found            | Is Rate Limit? (false)       | No Tweet Found / Other Error|                                                                                                        |
| No Tweet Found               | Set                  | Logs no tweet found status           | Is No Tweets? (true)         | -                          |                                                                                                        |
| Other Error                 | Set                  | Logs other unknown errors            | Is No Tweets? (false)        | -                          |                                                                                                        |
| Sticky Note                  | Sticky Note          | Overview and instructions            | -                           | -                          | This workflow automates your X (Twitter) engagement. It runs on a schedule, searches for new tweets based on a query, and automatically sends a reply. |
| Sticky Note2                 | Sticky Note          | Section 1 description                 | -                           | -                          | ## 1. Search for Tweets This section runs on a schedule and fetches new tweets based on your query.     |
| Sticky Note1                 | Sticky Note          | Section 2 description                 | -                           | -                          | ## 2. Process & Send Reply This section prepares and sends your defined reply to the new tweets.        |
| Sticky Note3                 | Sticky Note          | Section 3 description                 | -                           | -                          | ## 3. Error Handling This section catches common errors like API rate limits or empty search results.    |

---

### 4. Reproducing the Workflow from Scratch

To recreate this workflow manually in n8n, follow these steps:

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 30 minutes (`minutesInterval` = 30).  
   - Name: `Schedule Trigger (15分ごと)`  

2. **Create HTTP Request Node to Search Tweets**  
   - Type: HTTP Request  
   - Name: `Search Tweets (HTTP)`  
   - URL: `https://api.twitter.com/2/tweets/search/recent`  
   - Method: GET (ensure method corresponds to API docs; default is GET)  
   - Query Parameters:  
     - `query`: e.g. `from:badassceo OR from:bozu_108 OR from:hirox246 OR from:takaichi_sanae -is:retweet`  
     - `tweet.fields`: `created_at,author_id`  
     - `expansions`: `author_id`  
     - `user.fields`: `username`  
     - `max_results`: `50`  
   - Authentication: Configure with Twitter OAuth2 credentials (Twitter Developer account required).  
   - Connect output of Schedule Trigger to this node.

3. **Create Code Node to Filter Latest Tweets Per Author**  
   - Type: Code  
   - Name: `Get Latest Tweet Per Account`  
   - Paste the provided JavaScript code that merges responses, handles errors, and filters tweets by author for latest only (code provided in workflow).  
   - Input: Connect from `Search Tweets (HTTP)` node.

4. **Create If Node to Check Success Status**  
   - Type: If  
   - Name: `Is Success?`  
   - Condition: Check if `$json.status === "success"`  
   - Input: Connect from `Get Latest Tweet Per Account`.

5. **Create Code Node to Prepare Reply Message**  
   - Type: Code  
   - Name: `Prepare Reply`  
   - Paste JavaScript code that extracts username, tweet ID, and sets fixed reply text:  
     `弊社の新サービスをご覧ください！ https://x.com/linepi_pi/status/1988478476761133063`  
   - Input: Connect from True output of `Is Success?`.

6. **Create HTTP Request Node to Send Reply Tweet**  
   - Type: HTTP Request  
   - Name: `Send Reply (HTTP)`  
   - URL: `https://api.twitter.com/2/tweets`  
   - Method: POST  
   - Body type: JSON  
   - Body content:  
     ```json
     {
       "text": {{$json["replyText"]}},
       "reply": { "in_reply_to_tweet_id": {{$json["replyToTweetId"]}} }
     }
     ```  
   - Authentication: Twitter OAuth2 credentials same as above.  
   - Enable "Continue On Fail" to catch errors without stopping the workflow.  
   - Input: Connect from `Prepare Reply`.

7. **Create If Node to Check for Duplicate Content Error**  
   - Type: If  
   - Name: `Is Duplicate?`  
   - Condition: Check if `$json.error` contains `"duplicate content"`  
   - Input: Connect from `Send Reply (HTTP)`.

8. **Create Set Node for Successful Reply Logging**  
   - Type: Set  
   - Name: `Reply Success`  
   - Assign fields: status `"✅ リプライ送信成功"`, replied tweet ID, reply time, reply text, username, original tweet, API response JSON.  
   - Input: Connect from False output of `Is Duplicate?`.

9. **Create Set Node for Duplicate Reply Skipped**  
   - Type: Set  
   - Name: `Already Replied (Skip)`  
   - Assign fields: status `"✅ 既にリプライ済み（スキップ）"`, reason explaining duplicate, tweet ID, username, checked time, original tweet, API message.  
   - Input: Connect from True output of `Is Duplicate?`.

10. **Create If Node to Check Rate Limit Error**  
    - Type: If  
    - Name: `Is Rate Limit?`  
    - Condition: Check if `$json.errorType === "rate_limit"`  
    - Input: Connect from False output of `Is Success?` (error branch).

11. **Create Set Node to Log Rate Limit Error**  
    - Type: Set  
    - Name: `Rate Limit Error`  
    - Assign fields: status `"⚠️ API制限エラー"`, reason "Twitter APIのレート制限に達しました。15分後に自動的に回復します。", error details, timestamp.  
    - Input: Connect from True output of `Is Rate Limit?`.

12. **Create If Node to Check No Tweets Status**  
    - Type: If  
    - Name: `Is No Tweets?`  
    - Condition: Check if `$json.status === "no_tweets"`  
    - Input: Connect from False output of `Is Rate Limit?`.

13. **Create Set Node to Log No Tweets Found**  
    - Type: Set  
    - Name: `No Tweet Found`  
    - Assign fields: status `"📭 ツイートなし"`, reason "監視対象ユーザーからのツイートが見つかりませんでした", timestamp.  
    - Input: Connect from True output of `Is No Tweets?`.

14. **Create Set Node to Log Other Errors**  
    - Type: Set  
    - Name: `Other Error`  
    - Assign fields: status `"❌ その他のエラー"`, error type, error message, timestamp.  
    - Input: Connect from False output of `Is No Tweets?`.

15. **(Optional) Create Set Node for Reply Error Logging**  
    - Type: Set  
    - Name: `Reply Error`  
    - Assign fields for logging reply send errors.  
    - This node is defined but not connected in the original workflow and can be integrated where needed.

16. **Add Sticky Note Nodes (Optional)**  
    - Add sticky notes with contents describing workflow sections and setup instructions as per original workflow.  
    - Position them for clarity near corresponding blocks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automates your X (Twitter) engagement. It runs on a schedule, searches for new tweets based on a query, and automatically sends a reply.                | Workflow overview sticky note                                                                       |
| Setup steps include adding Twitter OAuth2 credentials, defining a search query, and customizing the reply message in the "Prepare Reply" node.                       | Setup instructions in sticky note                                                                   |
| Uses Twitter API v2 endpoints: `tweets/search/recent` for searching tweets and `tweets` POST endpoint for posting replies.                                            | Official Twitter API v2 documentation: https://developer.twitter.com/en/docs/twitter-api             |
| Handles API rate limit errors by detecting error type and waiting for automatic recovery (15 minutes).                                                                | Error handling block explanation                                                                    |
| Prevents duplicate replies by checking error messages for "duplicate content" and skipping reposts.                                                                   | Duplicate reply prevention logic                                                                    |
| Predefined reply message is fixed text with a link, easily customizable in the `Prepare Reply` code node.                                                             | Customization point in code node                                                                     |
| The workflow expects an active Twitter Developer account with OAuth2 credentials configured in n8n for API authentication.                                            | Credential requirement                                                                               |
| Scheduled interval can be adjusted by changing the Schedule Trigger node interval configuration.                                                                       | Scheduling flexibility                                                                               |

---

This completes the comprehensive structured analysis and reconstruction guide for the "Scheduled Auto-replies to Targeted Tweets Using X (Twitter) API" workflow.