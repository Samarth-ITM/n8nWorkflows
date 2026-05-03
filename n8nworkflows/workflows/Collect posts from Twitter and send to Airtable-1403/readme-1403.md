Collect posts from Twitter and send to Airtable

https://n8nworkflows.xyz/workflows/collect-posts-from-twitter-and-send-to-airtable-1403


# Collect posts from Twitter and send to Airtable

### 1. Workflow Overview

This workflow automates collecting recent tweets matching a search query ("verstappen") from Twitter and appends new tweets to an Airtable base, preventing duplicates based on Tweet ID. It is designed for incremental data collection, where the first execution gathers up to 100 tweets, and subsequent runs add only tweets not already stored in Airtable.

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger initiates the workflow.
- **1.2 Twitter Data Retrieval:** Search Twitter for recent tweets based on a keyword.
- **1.3 Airtable Data Retrieval:** Fetch existing tweets from Airtable to identify duplicates.
- **1.4 Data Preparation:** Format Twitter and Airtable data to comparable structures.
- **1.5 Data Comparison:** Remove tweets already present in Airtable.
- **1.6 Data Storage:** Append only new tweets to Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Starts the workflow manually to control execution timing.
- **Nodes Involved:** 
  - On clicking 'execute'

- **Node Details:**

  - **On clicking 'execute'**
    - **Type:** Manual Trigger — initiates workflow execution on user command.
    - **Configuration:** No parameters; default manual trigger.
    - **Expressions/Variables:** None.
    - **Input Connections:** None (entry node).
    - **Output Connections:** Connects to Twitter and Airtable list nodes.
    - **Version Requirements:** None.
    - **Edge Cases:** None expected; user must manually trigger.
    - **Sub-workflow:** None.

#### 1.2 Twitter Data Retrieval

- **Overview:** Retrieves up to 100 tweets matching the search term "verstappen" from Twitter.
- **Nodes Involved:**
  - Twitter
  - set twitter data

- **Node Details:**

  - **Twitter**
    - **Type:** Twitter node (Search operation)
    - **Configuration:**
      - Operation: Search
      - Search Text: "verstappen"
      - Limit: 100 tweets (adjustable)
      - Result Type: Mixed (includes popular and recent tweets)
    - **Expressions/Variables:** None.
    - **Input Connections:** From Manual Trigger.
    - **Output Connections:** To "set twitter data".
    - **Version Requirements:** Twitter API credentials needed.
    - **Edge Cases:** Twitter API rate limiting, search errors, empty results.
    - **Sub-workflow:** None.

  - **set twitter data**
    - **Type:** Set node (data transformation)
    - **Configuration:**
      - Extracts fields from Twitter JSON:
        - Likes: favorite_count
        - Tweet: text
        - Tweet_id: id (numeric)
        - Tweet URL: constructed as `https://twitter.com/{screen_name}/status/{id_str}`
        - Author: in_reply_to_screen_name (note: this is the username the tweet replies to, not the author)
        - Time: created_at
      - Keeps only these fields.
    - **Expressions/Variables:**
      - Uses expressions to map JSON fields.
    - **Input Connections:** From Twitter node.
    - **Output Connections:** To "Leave only new tweets" node.
    - **Version Requirements:** None.
    - **Edge Cases:** Missing fields in tweets may cause nulls.
    - **Sub-workflow:** None.

#### 1.3 Airtable Data Retrieval

- **Overview:** Retrieves existing tweet records from Airtable to check for duplicates.
- **Nodes Involved:**
  - get airtable list
  - Set_AT_list

- **Node Details:**

  - **get airtable list**
    - **Type:** Airtable node (List operation)
    - **Configuration:**
      - Application: app36P08S3Jzki6qJ (Airtable app ID)
      - Table: tbl6rexxFBodzKVoC (Airtable table ID)
      - Operation: List all records
    - **Authentication:** Uses stored Airtable API credentials.
    - **Input Connections:** From Manual Trigger.
    - **Output Connections:** To Set_AT_list.
    - **Version Requirements:** Airtable API key valid.
    - **Edge Cases:** Airtable API limit, empty table, auth issues.
    - **Sub-workflow:** None.

  - **Set_AT_list**
    - **Type:** Set node
    - **Configuration:**
      - Extracts fields from Airtable records:
        - Likes
        - Tweet
        - Tweet_id
        - Tweet URL
        - Author
        - Time
      - Uses expressions to map from Airtable JSON fields.
      - Keeps only these fields.
    - **Expressions/Variables:** References `get airtable list` node JSON.
    - **Input Connections:** From get airtable list.
    - **Output Connections:** To "Leave only new tweets".
    - **Version Requirements:** None.
    - **Edge Cases:** Missing or null fields in Airtable records.
    - **Sub-workflow:** None.

#### 1.4 Data Comparison

- **Overview:** Compares Twitter data with Airtable data and filters out tweets already stored, avoiding duplicates.
- **Nodes Involved:**
  - Leave only new tweets

- **Node Details:**

  - **Leave only new tweets**
    - **Type:** Merge node (Remove key matches mode)
    - **Configuration:**
      - Mode: Remove items from first input whose `Tweet_id` matches any in second input.
      - Property names for comparison: `Tweet_id` in both inputs.
    - **Input Connections:** 
      - First input (Index 0): set twitter data (new tweets)
      - Second input (Index 1): Set_AT_list (existing Airtable tweets)
    - **Output Connections:** To "Append new tweets to airtable".
    - **Version Requirements:** None.
    - **Edge Cases:** If Tweet_id fields are missing or not matching type, filtering may fail.
    - **Sub-workflow:** None.

#### 1.5 Data Storage

- **Overview:** Appends filtered new tweets to Airtable.
- **Nodes Involved:**
  - Append new tweets to airtable

- **Node Details:**

  - **Append new tweets to airtable**
    - **Type:** Airtable node (Append operation)
    - **Configuration:**
      - Application: app36P08S3Jzki6qJ
      - Table: tbl6rexxFBodzKVoC
      - Operation: Append records
      - Add all fields from input
    - **Authentication:** Airtable API credentials.
    - **Input Connections:** From "Leave only new tweets".
    - **Output Connections:** None (end node).
    - **Version Requirements:** Airtable API key valid.
    - **Edge Cases:** API rate limits, schema mismatch, or invalid data types may cause errors.
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role               | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                                                     |
|--------------------------|--------------------|------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'     | Manual Trigger     | Starts the workflow manually | None                        | Twitter, get airtable list  |                                                                                                                                                                |
| Twitter                  | Twitter            | Searches tweets on Twitter   | On clicking 'execute'        | set twitter data            |                                                                                                                                                                |
| set twitter data          | Set                | Formats Twitter tweet data   | Twitter                     | Leave only new tweets        |                                                                                                                                                                |
| get airtable list         | Airtable           | Retrieves existing tweets    | On clicking 'execute'        | Set_AT_list                 |                                                                                                                                                                |
| Set_AT_list               | Set                | Formats Airtable records     | get airtable list           | Leave only new tweets        |                                                                                                                                                                |
| Leave only new tweets     | Merge              | Filters out existing tweets  | set twitter data, Set_AT_list | Append new tweets to airtable |                                                                                                                                                                |
| Append new tweets to airtable | Airtable       | Appends new tweets to Airtable | Leave only new tweets        | None                        |                                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**
   - Name: "On clicking 'execute'"
   - No parameters needed.

2. **Add a Twitter node:**
   - Name: "Twitter"
   - Credentials: Connect your Twitter API credentials.
   - Operation: "Search"
   - Search Text: "verstappen" (adjust as needed)
   - Limit: 100 (modifiable)
   - Additional Fields: Result Type = "mixed"
   - Connect input from "On clicking 'execute'"

3. **Add a Set node for Twitter data:**
   - Name: "set twitter data"
   - Keep only set enabled.
   - Add fields:
     - Number: Likes = `{{$node["Twitter"].json["favorite_count"]}}`
     - String: Tweet = `{{$node["Twitter"].json["text"]}}`
     - String: Tweet_id = `{{$node["Twitter"].json["id"]}}`
     - String: Tweet URL = `https://twitter.com/{{$node["Twitter"].json["user"]["screen_name"]}}/status/{{$node["Twitter"].json["id_str"]}}`
     - String: Author = `{{$node["Twitter"].json["in_reply_to_screen_name"]}}` (Note: this is the replied-to username)
     - String: Time = `{{$node["Twitter"].json["created_at"]}}`
   - Connect input from "Twitter"

4. **Add an Airtable node to list records:**
   - Name: "get airtable list"
   - Credentials: Connect your Airtable API credentials.
   - Application ID: Your Airtable app ID (e.g., "app36P08S3Jzki6qJ")
   - Table ID: Your table ID (e.g., "tbl6rexxFBodzKVoC")
   - Operation: List
   - Connect input from "On clicking 'execute'"

5. **Add a Set node for Airtable data:**
   - Name: "Set_AT_list"
   - Keep only set enabled.
   - Add fields (map from Airtable record fields):
     - Number: Likes = `{{$node["get airtable list"].json["fields"]["Likes"] || 0}}`
     - String: Tweet = `{{$node["get airtable list"].json["fields"]["Tweet"]}}`
     - String: Tweet_id = `{{$node["get airtable list"].json["fields"]["Tweet_id"]}}`
     - String: Tweet URL = `{{$node["get airtable list"].json["fields"]["Tweet URL"]}}`
     - String: Author = `{{$node["get airtable list"].json["fields"]["Author"]}}`
     - String: Time = `{{$node["get airtable list"].json["fields"]["Time"]}}`
   - Connect input from "get airtable list"

6. **Add a Merge node to filter new tweets:**
   - Name: "Leave only new tweets"
   - Mode: "Remove Key Matches"
   - Property Name 1: "Tweet_id"
   - Property Name 2: "Tweet_id"
   - Connect first input (Index 0) from "set twitter data"
   - Connect second input (Index 1) from "Set_AT_list"

7. **Add an Airtable node to append new tweets:**
   - Name: "Append new tweets to airtable"
   - Credentials: Use Airtable API credentials.
   - Application ID: Same as above
   - Table ID: Same as above
   - Operation: Append
   - Add All Fields: Enabled (true)
   - Connect input from "Leave only new tweets"

8. **Connect all nodes:**
   - "On clicking 'execute'" → "Twitter"
   - "On clicking 'execute'" → "get airtable list"
   - "Twitter" → "set twitter data"
   - "get airtable list" → "Set_AT_list"
   - "set twitter data" → "Leave only new tweets" (input 0)
   - "Set_AT_list" → "Leave only new tweets" (input 1)
   - "Leave only new tweets" → "Append new tweets to airtable"

9. **Test the workflow:**
   - Execute manually.
   - Observe Twitter tweets are fetched.
   - Existing tweets in Airtable are retrieved.
   - Only new tweets get appended.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow avoids duplicates by comparing Tweet IDs between Twitter data and Airtable records. | Best practice for incremental synchronization of social media data.                             |
| Twitter API credentials require proper authentication and API access level.                        | https://developer.twitter.com/en/docs/twitter-api                                          |
| Airtable API requires API key and correct application/table IDs.                                   | https://airtable.com/api                                                                        |
| The "Author" field uses "in_reply_to_screen_name" which indicates the username the tweet replies to, not the original tweet author. Consider adjusting if true author info is desired. |                                                                                                 |
| Result Type "mixed" in Twitter search returns both recent and popular tweets matching the query.   |                                                                                                 |

---

This detailed breakdown and reconstruction guide enable developers or AI agents to understand the workflow’s logic fully, anticipate common failure points (API limits, missing fields), and reproduce or customize the data collection and synchronization process between Twitter and Airtable effectively.