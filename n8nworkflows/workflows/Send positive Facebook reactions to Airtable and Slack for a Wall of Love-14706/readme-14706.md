Send positive Facebook reactions to Airtable and Slack for a Wall of Love

https://n8nworkflows.xyz/workflows/send-positive-facebook-reactions-to-airtable-and-slack-for-a-wall-of-love-14706


# Send positive Facebook reactions to Airtable and Slack for a Wall of Love

# 1. Workflow Overview

This workflow monitors recent Facebook Page posts on a schedule, retrieves reactions for each post, filters only positive reactions, stores those reactions in Airtable, and posts a celebratory message to Slack as a “Wall of Love”.

Its main use case is internal team visibility: positive audience engagement on Facebook is automatically captured and shared without manual checking. It is especially useful for marketing, community, or customer success teams that want lightweight morale signals and a simple historical log of positive feedback.

## 1.1 Scheduled Trigger and Page Setup
The workflow starts on a recurring schedule and sets the Facebook Page ID used in downstream API requests.

## 1.2 Fetch Recent Facebook Posts
It calls the Facebook Graph API to retrieve the latest posts from the configured page, then expands the returned `data` array into individual post items and iterates through them.

## 1.3 Fetch and Iterate Through Reactions
For each post, it calls the Facebook Graph API again to get reactions, splits the reactions array into individual items, and loops through them one by one.

## 1.4 Filter Positive Reactions
Each reaction is evaluated against an allowed set of positive reaction types: `LIKE`, `LOVE`, `WOW`, `HAHA`, and `CARE`.

## 1.5 Store and Notify
Positive reactions are written to Airtable and then sent to a Slack channel in a formatted motivational message.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Page Setup

**Overview:**  
This block defines when the workflow runs and establishes the Facebook Page ID used by the API calls. It is the workflow’s entry point and global setup layer.

**Nodes Involved:**  
- Cron – Check Facebook Reactions
- Set Facebook Page ID

### Node: Cron – Check Facebook Reactions
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a recurring interval.
- **Configuration choices:**  
  Uses the Schedule Trigger with an interval rule. The JSON does not specify a concrete interval value, so this should be considered incomplete until set in the editor.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Set Facebook Page ID`
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Misconfigured interval may cause overly frequent or missing runs.
  - Timezone misunderstandings may affect expected execution times.
- **Sub-workflow reference:**  
  None.

### Node: Set Facebook Page ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a workflow field named `page_id` for later use.
- **Configuration choices:**  
  Assigns a string field `page_id`, but the value is currently empty in the provided workflow. This must be filled with the target Facebook Page ID before successful execution.
- **Key expressions or variables used:**  
  - Output field: `page_id`
- **Input and output connections:**  
  - Input: `Cron – Check Facebook Reactions`
  - Output: `Fetch Facebook Page Posts`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Empty `page_id` will cause the Facebook posts request to fail or return an invalid endpoint response.
  - If the Page ID is incorrect, the Graph API may return permission or object-not-found errors.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Fetch Recent Facebook Posts

**Overview:**  
This block requests the latest posts from the Facebook Page and converts the response array into one item per post. It then loops over each post for reaction analysis.

**Nodes Involved:**  
- Fetch Facebook Page Posts
- Extract Posts List1
- Loop Through Posts

### Node: Fetch Facebook Page Posts
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Facebook Graph API to fetch recent posts from the page.
- **Configuration choices:**  
  - URL expression: `https://graph.facebook.com/v19.0/{{ $json.page_id }}/posts`
  - Query parameters:
    - `access_token`: currently blank, must be filled
    - `fields`: `id,message,created_time,permalink_url`
    - `limit`: `5`
- **Key expressions or variables used:**  
  - `{{ $json.page_id }}`
- **Input and output connections:**  
  - Input: `Set Facebook Page ID`
  - Output: `Extract Posts List1`
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Missing or expired Facebook access token.
  - Invalid or empty `page_id`.
  - Facebook Graph API permission issues.
  - Rate limiting by Meta.
  - Posts with missing `message` are possible; this workflow does not rely on `message`, so that is acceptable.
- **Sub-workflow reference:**  
  None.

### Node: Extract Posts List1
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the `data` array from the Facebook API response into separate items.
- **Configuration choices:**  
  - `fieldToSplitOut`: `data`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Fetch Facebook Page Posts`
  - Output: `Loop Through Posts`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - If the response has no `data` field or an empty array, no items continue.
  - API error responses that do not contain `data` will break the expected structure.
- **Sub-workflow reference:**  
  None.

### Node: Loop Through Posts
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through posts one item at a time and coordinates returning control after downstream processing.
- **Configuration choices:**  
  Default Split in Batches behavior, with no explicit batch size configured in the JSON.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Extract Posts List1`
  - Output 1: returns/continue path not used directly
  - Output 2: `Fetch Post Reactions`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Incorrect loop wiring can create infinite loops; here it is intentionally wired back from the reactions loop.
  - Empty upstream item list means no post iteration occurs.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Fetch and Iterate Through Reactions

**Overview:**  
For each post, this block retrieves reaction data and expands it into individual reaction items. It then loops through reactions so each one can be filtered and processed independently.

**Nodes Involved:**  
- Fetch Post Reactions
- Extract Reactions List
- Loop Through Reactions

### Node: Fetch Post Reactions
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Facebook Graph API to get reactions for the current post.
- **Configuration choices:**  
  - URL expression: `GET /v19.0/{{ $json.id }}/reactions`
  - Query parameters:
    - `fields`: `id,name,type`
    - `limit`: `50`
    - `access_token`: currently blank, must be filled
- **Key expressions or variables used:**  
  - `{{ $json.id }}`
- **Input and output connections:**  
  - Input: `Loop Through Posts`
  - Output: `Extract Reactions List`
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - The configured URL is unusual for a generic HTTP Request node because it starts with `GET /v19.0/...` rather than a full URL. In many n8n setups, this will fail unless a base URL is separately configured, which is not present in the JSON. In practice, this should likely be changed to `https://graph.facebook.com/v19.0/{{ $json.id }}/reactions`.
  - Missing/expired token.
  - Permission issues on reaction data.
  - Facebook may paginate reactions beyond 50; this workflow only fetches up to 50 per post and does not handle pagination.
- **Sub-workflow reference:**  
  None.

### Node: Extract Reactions List
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Converts the `data` array of reactions into individual items.
- **Configuration choices:**  
  - `fieldToSplitOut`: `data`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Fetch Post Reactions`
  - Output: `Loop Through Reactions`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Empty or missing `data` means no reaction items continue.
  - Nonstandard API error payloads will break the expected array split.
- **Sub-workflow reference:**  
  None.

### Node: Loop Through Reactions
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over reactions one by one and loops back after downstream handling.
- **Configuration choices:**  
  Default Split in Batches behavior.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Extract Reactions List`
  - Output 1: `Loop Through Posts`
  - Output 2: `Is Positive Reaction?`
  - Also receives loopback from `Post to Slack – Wall of Love`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - This looping design assumes downstream processing completes before requesting the next reaction.
  - If Slack or Airtable fails and no error handling exists, the loop may halt partway through a post’s reactions.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Filter Positive Reactions

**Overview:**  
This block checks each reaction against a whitelist of positive reaction types. Only matching reactions continue to storage and notification.

**Nodes Involved:**  
- Is Positive Reaction?

### Node: Is Positive Reaction?
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the current reaction type belongs to the accepted positive set.
- **Configuration choices:**  
  Uses a boolean expression as the left value and checks whether it is `true`.
- **Key expressions or variables used:**  
  ```js
  {{ ['LIKE','LOVE','WOW','HAHA','CARE'].includes($json.type) }}
  ```
- **Input and output connections:**  
  - Input: `Loop Through Reactions`
  - True output: `Save Reaction to Airtable`
  - False output: not connected
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If `type` is missing, the expression resolves to `false`.
  - Reaction type names are case-sensitive in this expression.
  - The sticky note text mentions Like, Love, Wow, and Haha, but the actual logic also includes `CARE`.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Store and Notify

**Overview:**  
This block persists each positive reaction in Airtable and then posts a formatted message to Slack. It is the final business-output stage of the workflow.

**Nodes Involved:**  
- Save Reaction to Airtable
- Post to Slack – Wall of Love

### Node: Save Reaction to Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Creates a new record in Airtable for each positive reaction.
- **Configuration choices:**  
  - Operation: `create`
  - Base: `Fake Review Detector`
  - Table: `FacebookReactionData`
  - Field mapping:
    - `Id` ← `{{ $json.id }}`
    - `type` ← `{{ $json.type }}`
    - `username` ← `{{ $json.name }}`
- **Key expressions or variables used:**  
  - `{{ $json.id }}`
  - `{{ $json.type }}`
  - `{{ $json.name }}`
- **Input and output connections:**  
  - Input: `Is Positive Reaction?`
  - Output: `Post to Slack – Wall of Love`
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Airtable authentication failure.
  - Table schema mismatch if fields `Id`, `type`, or `username` do not exist or are renamed.
  - Duplicate reactions are not prevented; repeated workflow runs can create duplicate rows because no matching/upsert logic is used.
- **Sub-workflow reference:**  
  None.

### Node: Post to Slack – Wall of Love
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a celebratory text message into a specified Slack channel.
- **Configuration choices:**  
  - Send mode: channel
  - Channel: `n8n`
  - Link to workflow disabled
  - Message text uses Airtable response fields:
    - `{{ $json.fields.username }}`
    - conditional formatting for `{{ $json.fields.type }}`
- **Key expressions or variables used:**  
  ```js
  =🔥 New Positive Reaction!

  👤 {{ $json.fields.username }}
  {{ 
    $json.fields.type === 'LOVE' ? '❤️ LOVE' :
    $json.fields.type === 'WOW'  ? '😮 WOW'  :
    $json.fields.type === 'LIKE' ? '👍 LIKE' :
    $json.fields.type === 'HAHA' ? '😂 HAHA' :
    '💙 CARE'
  }}

  Keep up the great work, team! 🚀
  ```
- **Input and output connections:**  
  - Input: `Save Reaction to Airtable`
  - Output: `Loop Through Reactions`
- **Version-specific requirements:**  
  Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Slack auth or channel permission issues.
  - If Airtable changes its response shape, `$json.fields.*` may not resolve as expected.
  - Any reaction type outside the listed explicit checks defaults visually to `💙 CARE`, though the filter currently ensures only approved types pass.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Cron – Check Facebook Reactions | Schedule Trigger | Starts the workflow on a recurring schedule |  | Set Facebook Page ID | ## Schedule & Setup<br>**This section defines when the workflow runs and which Facebook Page it should monitor. The scheduler triggers the automation at regular intervals and the Page ID is set once so it can be reused across the workflow.** |
| Set Facebook Page ID | Set | Stores the target Facebook Page ID in workflow data | Cron – Check Facebook Reactions | Fetch Facebook Page Posts | ## Schedule & Setup<br>**This section defines when the workflow runs and which Facebook Page it should monitor. The scheduler triggers the automation at regular intervals and the Page ID is set once so it can be reused across the workflow.** |
| Fetch Facebook Page Posts | HTTP Request | Retrieves recent posts from the Facebook Page via Graph API | Set Facebook Page ID | Extract Posts List1 | ## Fetch Facebook Posts<br>**This section retrieves the latest posts from the Facebook Page and processes them one by one. Each post is prepared individually so reactions can be checked accurately.** |
| Extract Posts List1 | Split Out | Splits the posts array into individual post items | Fetch Facebook Page Posts | Loop Through Posts | ## Fetch Facebook Posts<br>**This section retrieves the latest posts from the Facebook Page and processes them one by one. Each post is prepared individually so reactions can be checked accurately.** |
| Loop Through Posts | Split In Batches | Iterates through posts sequentially | Extract Posts List1, Loop Through Reactions | Fetch Post Reactions | ## Fetch Facebook Posts<br>**This section retrieves the latest posts from the Facebook Page and processes them one by one. Each post is prepared individually so reactions can be checked accurately.** |
| Fetch Post Reactions | HTTP Request | Retrieves reactions for the current post | Loop Through Posts | Extract Reactions List | ## Fetch & Process Reactions<br>**This section collects all reactions for each post and breaks them down into individual reactions. Processing reactions one by one allows precise filtering and storage.** |
| Extract Reactions List | Split Out | Splits the reactions array into individual reaction items | Fetch Post Reactions | Loop Through Reactions | ## Fetch & Process Reactions<br>**This section collects all reactions for each post and breaks them down into individual reactions. Processing reactions one by one allows precise filtering and storage.** |
| Loop Through Reactions | Split In Batches | Iterates through reactions sequentially and coordinates loop progression | Extract Reactions List, Post to Slack – Wall of Love | Loop Through Posts, Is Positive Reaction? | ## Fetch & Process Reactions<br>**This section collects all reactions for each post and breaks them down into individual reactions. Processing reactions one by one allows precise filtering and storage.** |
| Is Positive Reaction? | If | Filters only positive Facebook reaction types | Loop Through Reactions | Save Reaction to Airtable | ## Filter Positive Reactions<br>**This section filters out only positive reactions such as Like, Love, Wow and Haha. Negative or neutral reactions are ignored so the workflow focuses purely on motivation and appreciation.** |
| Save Reaction to Airtable | Airtable | Stores positive reaction details in Airtable | Is Positive Reaction? | Post to Slack – Wall of Love | ## Store & Notify Team<br>**This final section stores positive reactions for tracking and notifies the team on Slack. Each reaction appears as a small morale boost, helping create a visible “Wall of Love”.** |
| Post to Slack – Wall of Love | Slack | Sends a formatted team notification to Slack | Save Reaction to Airtable | Loop Through Reactions | ## Store & Notify Team<br>**This final section stores positive reactions for tracking and notifies the team on Slack. Each reaction appears as a small morale boost, helping create a visible “Wall of Love”.** |
| Sticky Note | Sticky Note | Documentation note for schedule and setup section |  |  |  |
| Sticky Note1 | Sticky Note | Documentation note for Facebook posts retrieval section |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note for reaction processing section |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note for positive reaction filtering section |  |  |  |
| Sticky Note4 | Sticky Note | Documentation note for storage and Slack notification section |  |  |  |
| Sticky Note5 | Sticky Note | General explanatory note and setup guidance |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Facebook Reactions to Airtable and Slack Wall of Love`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Cron – Check Facebook Reactions`
   - Configure an interval suitable for your needs, for example every hour or every day.
   - This is the workflow entry point.

3. **Add a Set node**
   - Node type: **Set**
   - Name: `Set Facebook Page ID`
   - Add one field:
     - `page_id` as **String**
     - Value: your Facebook Page ID
   - Connect:
     - `Cron – Check Facebook Reactions` → `Set Facebook Page ID`

4. **Add an HTTP Request node to fetch posts**
   - Node type: **HTTP Request**
   - Name: `Fetch Facebook Page Posts`
   - Method: `GET`
   - URL:
     - `https://graph.facebook.com/v19.0/{{ $json.page_id }}/posts`
   - Enable query parameters:
     - `access_token`: your Facebook Graph API access token
     - `fields`: `id,message,created_time,permalink_url`
     - `limit`: `5`
   - Connect:
     - `Set Facebook Page ID` → `Fetch Facebook Page Posts`

5. **Add a Split Out node for posts**
   - Node type: **Split Out**
   - Name: `Extract Posts List1`
   - Field to split out:
     - `data`
   - Connect:
     - `Fetch Facebook Page Posts` → `Extract Posts List1`

6. **Add a Split In Batches node for posts**
   - Node type: **Split In Batches**
   - Name: `Loop Through Posts`
   - Keep default batch behavior unless you need a specific size.
   - Connect:
     - `Extract Posts List1` → `Loop Through Posts`

7. **Add an HTTP Request node to fetch reactions**
   - Node type: **HTTP Request**
   - Name: `Fetch Post Reactions`
   - Method: `GET`
   - Recommended URL:
     - `https://graph.facebook.com/v19.0/{{ $json.id }}/reactions`
   - Query parameters:
     - `fields`: `id,name,type`
     - `limit`: `50`
     - `access_token`: your Facebook Graph API access token
   - Important: use a full URL. The JSON provided uses `GET /v19.0/...`, which is likely not portable and should be corrected.
   - Connect:
     - `Loop Through Posts` → `Fetch Post Reactions`

8. **Add a Split Out node for reactions**
   - Node type: **Split Out**
   - Name: `Extract Reactions List`
   - Field to split out:
     - `data`
   - Connect:
     - `Fetch Post Reactions` → `Extract Reactions List`

9. **Add a Split In Batches node for reactions**
   - Node type: **Split In Batches**
   - Name: `Loop Through Reactions`
   - Connect:
     - `Extract Reactions List` → `Loop Through Reactions`

10. **Wire the nested loop structure**
    - Connect `Loop Through Reactions` main processing output to `Is Positive Reaction` later.
    - Connect the completion/return path so reaction processing can continue and eventually return to the post loop:
      - `Loop Through Reactions` → `Loop Through Posts`
      - `Post to Slack – Wall of Love` → `Loop Through Reactions`
    - In practice, preserve the same loop pattern as the JSON so reactions are processed before advancing through posts.

11. **Add an If node for positive filtering**
    - Node type: **If**
    - Name: `Is Positive Reaction?`
    - Configure a boolean condition using an expression:
      - `{{ ['LIKE','LOVE','WOW','HAHA','CARE'].includes($json.type) }}`
    - Compare that expression as `true`.
    - Connect:
      - `Loop Through Reactions` → `Is Positive Reaction?`

12. **Create the Airtable destination**
    - In Airtable, prepare:
      - A base
      - A table, e.g. `FacebookReactionData`
    - Add at least these fields:
      - `Id` as single line text
      - `username` as single line text
      - `type` as single line text

13. **Add Airtable credentials in n8n**
    - Use an **Airtable Personal Access Token** credential.
    - Ensure the token has access to the target base and table.

14. **Add the Airtable node**
    - Node type: **Airtable**
    - Name: `Save Reaction to Airtable`
    - Resource: record creation
    - Operation: `Create`
    - Select your base and table
    - Map fields:
      - `Id` → `{{ $json.id }}`
      - `username` → `{{ $json.name }}`
      - `type` → `{{ $json.type }}`
    - Connect:
      - `Is Positive Reaction` true output → `Save Reaction to Airtable`

15. **Add Slack credentials**
    - Use a Slack API credential in n8n.
    - The Slack app must be authorized to post to the target channel.

16. **Add the Slack notification node**
    - Node type: **Slack**
    - Name: `Post to Slack – Wall of Love`
    - Operation: send a message to a channel
    - Select the target channel
    - Message text:
      ```text
      🔥 New Positive Reaction!

      👤 {{ $json.fields.username }}
      {{ 
        $json.fields.type === 'LOVE' ? '❤️ LOVE' :
        $json.fields.type === 'WOW'  ? '😮 WOW'  :
        $json.fields.type === 'LIKE' ? '👍 LIKE' :
        $json.fields.type === 'HAHA' ? '😂 HAHA' :
        '💙 CARE'
      }}

      Keep up the great work, team! 🚀
      ```
    - Disable “Include link to workflow” if desired.
    - Connect:
      - `Save Reaction to Airtable` → `Post to Slack – Wall of Love`

17. **Close the reaction loop**
    - Connect:
      - `Post to Slack – Wall of Love` → `Loop Through Reactions`
    - This allows the next reaction to be processed.

18. **Close the post loop**
    - Connect:
      - `Loop Through Reactions` → `Loop Through Posts`
    - This returns control to the post iterator after reaction processing completes.

19. **Add optional sticky notes for maintainability**
    - You can reproduce the notes from the original workflow:
      - Schedule & Setup
      - Fetch Facebook Posts
      - Fetch & Process Reactions
      - Filter Positive Reactions
      - Store & Notify Team
      - General workflow explanation and setup steps

20. **Test the Facebook API first**
    - Run the workflow manually.
    - Confirm:
      - The page posts response contains `data`
      - The reaction response contains `data`
      - Your token has sufficient Facebook permissions

21. **Test Airtable writes**
    - Verify a positive reaction creates a record with:
      - `Id`
      - `username`
      - `type`

22. **Test Slack output**
    - Confirm the Slack node posts into the correct channel.
    - Verify the message renders the expected emoji label.

23. **Consider production hardening**
    - Add deduplication before Airtable if you do not want the same reaction recorded repeatedly across scheduled runs.
    - Add pagination if you need more than 5 posts or more than 50 reactions.
    - Add error handling branches for Facebook, Airtable, and Slack failures.

24. **Activate the workflow**
    - Once validated, enable the workflow so the schedule runs automatically.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically monitors positive reactions on Facebook posts and shares them with the team to boost morale. It runs on a scheduled basis, checks the selected Facebook Page for new posts and processes each post to capture reactions. | General workflow purpose |
| Only positive reactions such as Like, Love, Wow and Haha are considered, while negative reactions are ignored. Each positive reaction is saved in Airtable for tracking and then shared in a Slack “Wall of Love” channel, helping teams stay connected to real-time positive customer feedback without any manual effort. | General workflow behavior |
| Configure the scheduler to define how often the workflow should check for new Facebook reactions. | Setup guidance |
| Set the Facebook Page ID so the workflow knows which page to monitor. | Setup guidance |
| Connect the Facebook API (or mock data during testing) to fetch posts and reactions. | Setup guidance |
| Add filtering logic to allow only positive reactions to pass through. | Setup guidance |
| Connect Airtable and map the fields to store reaction details properly. | Setup guidance |
| Configure the Slack node with the target channel and message format. | Setup guidance |
| Test the workflow using mock data to confirm messages appear correctly in Slack. | Setup guidance |
| Enable the workflow to start automatic monitoring. | Setup guidance |

## Additional implementation notes
- The workflow has **one entry point**: `Cron – Check Facebook Reactions`.
- The workflow has **no sub-workflow nodes** and does not invoke any child workflow.
- The provided JSON contains **blank Facebook credentials/values** for:
  - `page_id`
  - `access_token` in the posts request
  - `access_token` in the reactions request
- The `Fetch Post Reactions` node should preferably use a full Graph API URL to avoid environment-specific failures.
- There is **no deduplication logic**, so repeated scheduled runs may insert the same reactions into Airtable and post duplicate Slack messages.
- There is **no pagination handling**, so only the latest 5 posts and up to 50 reactions per post are processed per run.