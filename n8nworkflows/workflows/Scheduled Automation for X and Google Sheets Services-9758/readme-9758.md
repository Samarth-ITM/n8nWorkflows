Scheduled Automation for X and Google Sheets Services

https://n8nworkflows.xyz/workflows/scheduled-automation-for-x-and-google-sheets-services-9758


# Scheduled Automation for X and Google Sheets Services

### 1. Workflow Overview

This workflow automates the process of engaging with Twitter (X) accounts listed in a Google Sheet by liking and reposting their latest tweets on a scheduled basis. It is designed for social media managers or automation specialists who want to maintain regular interaction with a curated list of X accounts without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Automatically initiates the workflow at a set time daily.
- **1.2 Data Retrieval from Google Sheets:** Reads the list of target Twitter accounts from a specified Google Sheet.
- **1.3 Twitter Query for Latest Tweets:** Searches each listed account’s latest tweets, avoiding replies and retweets.
- **1.4 Action Limiting:** Caps the number of like and repost actions per run to comply with API rate limits and reduce spam risk.
- **1.5 Like Tweets:** Likes the tweets identified in the previous step.
- **1.6 Repost Tweets:** Reposts (retweets) the tweets just liked.

Supporting sticky notes provide guidance, best practices, and configuration tips for each functional block.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically on a defined schedule, here configured to trigger daily at 19:00 (7 PM).

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Configuration: Set to trigger once daily at 19:00 hours.  
    - Input: None (start node)  
    - Output: Triggers the "Read accounts from Google Sheets" node.  
    - Edge Cases:  
      - If the schedule overlaps with API rate limits, it may cause failures or throttling.  
      - Manual execution possible for testing.  
    - Notes: Sticky note advises running off-peak hours and testing manually during development.

#### 2.2 Data Retrieval from Google Sheets

- **Overview:**  
  Reads a list of Twitter account screen names (without '@') from a Google Sheet document to determine which accounts to process.

- **Nodes Involved:**  
  - Read accounts from Google Sheets

- **Node Details:**  
  - **Read accounts from Google Sheets**  
    - Type: `Google Sheets`  
    - Configuration:  
      - Document ID and Sheet Name (`gid=0`) specified to locate the sheet.  
      - Reads the column `account_id` which contains Twitter screen names.  
    - Input: Triggered by Schedule Trigger.  
    - Output: Passes account list items individually to "Get latest tweets" node.  
    - Edge Cases:  
      - Incorrect or missing document ID or sheet name causes failure.  
      - Missing or misspelled `account_id` column leads to empty or invalid data.  
    - Notes: Sticky note suggests optional additional columns (`blocked_words`, `last_processed_at`) for future extensions.

#### 2.3 Twitter Query for Latest Tweets

- **Overview:**  
  For each account ID read from the sheet, fetches the latest tweets excluding replies and retweets.

- **Nodes Involved:**  
  - Get latest tweets

- **Node Details:**  
  - **Get latest tweets**  
    - Type: `Twitter` node  
    - Configuration:  
      - Operation: Search  
      - Search Text: `from:{{ $json['account_id'] }} -is:retweet -is:reply`  
      - Limit: 10 tweets per account  
      - No additional tweet fields configured, but can be extended.  
    - Input: Account data from Google Sheets node.  
    - Output: Sends tweet details downstream for limiting.  
    - Edge Cases:  
      - API rate limits if many accounts or tweets requested simultaneously.  
      - Empty results if accounts are inactive or search query incorrect.  
      - Authentication errors if Twitter OAuth2 credentials expire.  
    - Notes: Sticky note recommends filtering for freshness and avoiding rate limits.

#### 2.4 Action Limiting

- **Overview:**  
  Limits the number of tweets to be liked and reposted in a single run to prevent spam flags and API overuse.

- **Nodes Involved:**  
  - Limit

- **Node Details:**  
  - **Limit**  
    - Type: `Limit`  
    - Configuration: No explicit limit set in JSON; recommended production limits are 1–3 actions per run.  
    - Input: Tweets from "Get latest tweets."  
    - Output: Limited set of tweets forwarded to "Like tweet."  
    - Edge Cases:  
      - If limit is too low, engagement will be minimal.  
      - If limit is too high, risk of account suspension due to spam behavior.  
    - Notes: Sticky note stresses starting with low values and possibly varying limits by time/day.

#### 2.5 Like Tweets

- **Overview:**  
  Likes each tweet passed from the limit node. Supports a dry-run toggle externally for testing without real actions.

- **Nodes Involved:**  
  - Like tweet

- **Node Details:**  
  - **Like tweet**  
    - Type: `Twitter`  
    - Configuration:  
      - Operation: Like  
      - Tweet ID sourced from the previous node’s tweet data (`{{$json.id}}`).  
    - Input: Limited tweets.  
    - Output: Forwards liked tweet data to "Repost tweet."  
    - Edge Cases:  
      - Tweets already liked may cause errors or no-ops.  
      - Network or auth errors may cause failure.  
      - Dry-run mode requires additional nodes (Set + IF) outside this workflow.  
    - Notes: Sticky note recommends filtering tweets for blocked words or freshness before like.

#### 2.6 Repost Tweets

- **Overview:**  
  Retweets the tweets that were just liked, completing the engagement cycle.

- **Nodes Involved:**  
  - Repost tweet

- **Node Details:**  
  - **Repost tweet**  
    - Type: `Twitter`  
    - Configuration:  
      - Operation: Retweet  
      - Tweet ID sourced from previous node (`{{$json.id}}`).  
    - Input: From Like tweet node.  
    - Output: None (end node).  
    - Edge Cases:  
      - Retweeting a tweet already retweeted may cause errors.  
      - API rate limits and cooldowns should be managed externally or via additional logic.  
    - Notes: Sticky note suggests adding cooldowns or time gates for brand safety.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                      | Input Node(s)                | Output Node(s)           | Sticky Note                                                                                   |
|------------------------------|--------------------|------------------------------------|------------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger   | Automatic workflow start on schedule | None                         | Read accounts from Google Sheets | Cadence: hourly/daily; test with manual execution; avoid peak hours                          |
| Read accounts from Google Sheets | Google Sheets     | Reads target Twitter accounts       | Schedule Trigger             | Get latest tweets          | Required column `account_id`; replace documentId/sheetName; optional columns for future use  |
| Get latest tweets            | Twitter            | Fetches latest tweets per account    | Read accounts from Google Sheets | Limit                    | Query excludes replies/retweets; limit 1–5 tweets; watch rate limits                        |
| Limit                       | Limit              | Caps number of like/repost actions   | Get latest tweets            | Like tweet                | Recommended 1–3 actions; helps avoid spam flags; vary by time/day if needed                   |
| Like tweet                  | Twitter            | Likes tweets                         | Limit                       | Repost tweet              | Use dry-run flag externally; filter blocked words/freshness before like                      |
| Repost tweet                | Twitter            | Retweets tweets just liked           | Like tweet                  | None                      | Add cooldown/time-of-day gates; keep repost filters strict                                  |
| 🟨 Sticky: Overview          | Sticky Note        | Overview and instructions             | None                         | None                      | Purpose, requirements, flow, tips/security summarized                                       |
| Sticky: Schedule Trigger    | Sticky Note        | Schedule trigger usage info           | None                         | None                      | Schedule cadence, testing instructions                                                     |
| Sticky: Google Sheets Read  | Sticky Note        | Google Sheets read guidance           | None                         | None                      | Required columns, document/sheet ID, header options                                        |
| Sticky: Get Latest Tweets   | Sticky Note        | Twitter search tips                   | None                         | None                      | Query example, limits, rate limit notes                                                    |
| Sticky: Limit               | Sticky Note        | Limit node best practices             | None                         | None                      | Recommended caps, spam avoidance, time-based variation                                    |
| Sticky: Like Tweet          | Sticky Note        | Like tweet node best practices        | None                         | None                      | Dry-run, safety filters                                                                    |
| Sticky: Repost Tweet        | Sticky Note        | Repost tweet node best practices      | None                         | None                      | Cooldown and brand safety recommendations                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 19:00 (7 PM) or adjust as needed for rate limits.  
   - No input connections.

2. **Create the Google Sheets node:**  
   - Type: Google Sheets  
   - Configure OAuth2 credentials for Google API access.  
   - Set Document ID to your Google Sheet containing the account list.  
   - Set Sheet Name to the appropriate tab (e.g., `gid=0`).  
   - Ensure the sheet contains a column named `account_id` with Twitter screen names (without '@').  
   - Connect the Schedule Trigger’s output to this node.

3. **Create the Twitter node to get latest tweets:**  
   - Type: Twitter  
   - Configure OAuth2 credentials for Twitter (X) API access.  
   - Operation: Search  
   - Search Text: Use dynamic expression: `from:{{ $json['account_id'] }} -is:retweet -is:reply`  
   - Limit: 10 tweets per account (adjust as needed).  
   - Connect the Google Sheets node output to this node.

4. **Create the Limit node:**  
   - Type: Limit  
   - Set a maximum number of items to pass downstream (recommended 1–3).  
   - Connect the Twitter search node output to this node.

5. **Create the Twitter node to Like tweets:**  
   - Type: Twitter  
   - Operation: Like  
   - Tweet ID: Use expression `={{ $json.id }}` to dynamically like each tweet.  
   - Connect the Limit node output to this node.  
   - Optional: Implement a dry-run by adding a Set node with a flag and an IF node before this step (not in base workflow).

6. **Create the Twitter node to Repost tweets:**  
   - Type: Twitter  
   - Operation: Retweet  
   - Tweet ID: Use expression `={{ $json.id }}` from the Like tweet node output.  
   - Connect the Like tweet node output to this node.

7. **Add Sticky Notes for clarity:**  
   - Create sticky notes near each logical section with the content from the original sticky notes for documentation and guidance.  
   - Include overview, scheduling tips, Google Sheets guidance, Twitter query tips, limit best practices, and safety measures for liking and reposting.

8. **Set Credentials:**  
   - Ensure Google Sheets node uses valid Google OAuth2 credentials with read access to the specified document.  
   - Ensure Twitter nodes use valid OAuth2 credentials with permissions for reading tweets, liking, and retweeting.

9. **Test the workflow:**  
   - Run manually first to verify data flow and credentials.  
   - Adjust limits and schedule to avoid hitting API rate limits or spam flags.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| Never hardcode API tokens or credentials inside nodes; always use n8n’s credential management for security.                                                        | Security best practices                                               |
| Respect API rate limits of both Twitter (X) and Google Sheets APIs by scheduling appropriately and limiting actions per run.                                      | API usage guidelines                                                  |
| For safety, implement a dry-run mode (using Set and IF nodes) during development to avoid unintended likes or retweets.                                           | Workflow testing advice                                               |
| Consider adding filters for blocked words, language, or tweet freshness to reduce brand risk and improve relevance.                                               | Suggested future extensions                                           |
| Sticky notes provide valuable operational tips and reminders; include them visibly in your workflow for clarity.                                                  | Workflow documentation practice                                       |
| Use official n8n documentation for configuring OAuth2 credentials for Twitter and Google Sheets.                                                                   | https://docs.n8n.io/credentials/oauth2/                               |
| Twitter search syntax reference: https://developer.twitter.com/en/docs/twitter-api/tweets/search/integrate/build-a-query                                         | Twitter API documentation                                             |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.