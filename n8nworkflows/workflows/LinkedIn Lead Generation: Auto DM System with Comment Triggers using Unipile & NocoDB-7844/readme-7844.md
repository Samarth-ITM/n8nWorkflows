LinkedIn Lead Generation: Auto DM System with Comment Triggers using Unipile & NocoDB

https://n8nworkflows.xyz/workflows/linkedin-lead-generation--auto-dm-system-with-comment-triggers-using-unipile---nocodb-7844


# LinkedIn Lead Generation: Auto DM System with Comment Triggers using Unipile & NocoDB

### 1. Workflow Overview

This workflow automates lead generation on LinkedIn by monitoring comments on specified posts, filtering for trigger words, and sending direct messages (DMs) or connection requests accordingly. It leverages Unipile’s API to interact with LinkedIn data, NocoDB as a backend database to track message and connection statuses, and incorporates randomized messaging and timing to simulate natural human behavior.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user inputs from a form, including the LinkedIn post ID, trigger word, and lead magnet link.
- **1.2 Comment Retrieval & Filtering:** Fetches comments from the specified LinkedIn post, paginates through them, and filters those containing the trigger word.
- **1.3 Connection Status Check & Filtering:** Checks the connection level of commenters and filters out those already contacted or requested via NocoDB.
- **1.4 Messaging Logic:** Depending on connection status, either sends a DM with the lead magnet or replies asking for a connection request.
- **1.5 Logging & Throttling:** Logs interactions in NocoDB, manages message rotation for natural replies, and throttles message sending with randomized wait times.
- **1.6 Pagination Handling & Looping:** Handles pagination of comment fetching and loops over batched comment processing to cover all relevant commenters.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures the workflow’s trigger inputs from a form submission, including the LinkedIn post ID, trigger word, and lead magnet URL.
- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (Settings)

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point, captures user input fields: Post ID (required), Trigger word, Lead Magnet Link  
    - *Key Variables:* `Post ID`, `Trigger word`, `Lead Magnet Link` accessible via expressions e.g. `$('On form submission').first().json["Post ID"]`  
    - *Connections:* Outputs to `cursor` initialization node  
    - *Potential Failures:* Form webhook failures or missing required fields  
    - *Sticky Note Content:* Describes inputs: post ID, trigger, lead magnet URL  

---

#### 2.2 Comment Retrieval & Filtering

- **Overview:** Retrieves all comments from the specified LinkedIn post (using Unipile API), handles pagination, and filters comments containing the trigger word.
- **Nodes Involved:**  
  - cursor (Code)  
  - Get Comments from Posts (HTTP Request)  
  - Split Out1 (Split Out)  
  - Filter Comments with trigger (Filter)  
  - Sticky Note1  

- **Node Details:**

  - **cursor**  
    - *Type:* Code  
    - *Role:* Extracts pagination cursor from previous comments fetch to enable iterative fetching  
    - *Key Expression:* Attempts to read `cursor` from `Get Comments from Posts` output or defaults to empty string  
    - *Connections:* Feeds `cursor` as query parameter to `Get Comments from Posts`  
    - *Potential Failures:* Missing cursor property leading to empty pagination  

  - **Get Comments from Posts**  
    - *Type:* HTTP Request  
    - *Role:* Calls Unipile API endpoint to get comments for the specified LinkedIn post, with pagination support  
    - *Key Config:* URL dynamically built from form input Post ID; passes account ID and cursor as query parameters; uses HTTP header authentication via UNIPILE credentials  
    - *Connections:* Outputs to `Split Out1`  
    - *Failures:* API auth errors, rate limits, network timeouts  

  - **Split Out1**  
    - *Type:* Split Out  
    - *Role:* Extracts the `items` array from comments response for further processing  
    - *Connections:* Outputs to `Filter Comments with trigger`  

  - **Filter Comments with trigger**  
    - *Type:* Filter  
    - *Role:* Filters comments whose text contains the trigger word (case insensitive), using expression referencing form input  
    - *Key Expression:* Checks if `text.toLowerCase()` contains `Trigger word` from form  
    - *Connections:* Outputs to `Check if we are connected` node  

  - *Sticky Note1:* Explains this block fetches all comments and filters by trigger word  

---

#### 2.3 Connection Status Check & Filtering

- **Overview:** Determines if commenters are 1st-degree connections and filters out those already contacted or requested, avoiding duplicate outreach.
- **Nodes Involved:**  
  - Check if we are connected (If)  
  - Get all the rows where the dm is already sent (NocoDB)  
  - Get all the rows where the connection is already asked (NocoDB)  
  - Filter people where we have not sent the DM yet (Code)  
  - Filter people where we have not asked connexion yet (Code)  
  - If1 (If)  
  - If (If)  
  - Split Out2 (Split Out)  
  - Split Out (Split Out)  
  - Sticky Note2, Sticky Note3  

- **Node Details:**

  - **Check if we are connected**  
    - *Type:* If  
    - *Role:* Checks if commenter’s `network_distance` equals `DISTANCE_1` (1st degree connection)  
    - *Connections:* True branch (connected) leads to DM logic; False branch leads to connection request logic  
    - *Failures:* Missing or malformed connection data  

  - **Get all the rows where the dm is already sent**  
    - *Type:* NocoDB (Get All)  
    - *Role:* Fetches all entries where `dm_status` = `sent` to track who has received DMs  
    - *Connections:* Feeds into code node filtering those not yet messaged  
    - *Failures:* API auth errors, data inconsistencies  

  - **Get all the rows where the connection is already asked**  
    - *Type:* NocoDB (Get All)  
    - *Role:* Fetches entries with `dm_status` = `connection request` to track connection requests sent  
    - *Connections:* Feeds into code node filtering those not yet requested  
    - *Failures:* Same as above  

  - **Filter people where we have not sent the DM yet**  
    - *Type:* Code  
    - *Role:* Filters commenters connected at 1st degree but not in the 'DM sent' list  
    - *Key Logic:* Compares LinkedIn IDs from commenters and database records  
    - *Connections:* Outputs filtered array to `If1` node  

  - **Filter people where we have not asked connexion yet**  
    - *Type:* Code  
    - *Role:* Filters commenters not connected (or connection not 1st degree) and not already asked for connection, excluding those with reply_counter > 0  
    - *Connections:* Outputs filtered array to `If` node  

  - **If1 and If**  
    - *Type:* If  
    - *Role:* Check if filtered lists are not empty to continue processing  
    - *Connections:* True branches lead to respective Split Out nodes  

  - **Split Out2 & Split Out**  
    - *Type:* Split Out  
    - *Role:* Extract `filtered` array elements to process individually  
    - *Connections:* Split Out2 leads to `Loop Over Items1` for DM sending; Split Out leads to `Ask For connection`  

  - *Sticky Note2:* Describes connection level check and decision to DM or request connection  
  - *Sticky Note3:* Notes filtering to prevent duplicate messaging or connection requests  

---

#### 2.4 Messaging Logic

- **Overview:** Sends personalized DMs or connection request comments depending on connection status, using randomized timing and message rotation to mimic human behavior.
- **Nodes Involved:**  
  - Loop Over Items1 (Split In Batches)  
  - Wait (Wait)  
  - Send DM with Lead Magnet (HTTP Request)  
  - Create Message Rotation (Code)  
  - Answer to comment (HTTP Request)  
  - Ask For connection (HTTP Request)  
  - Create a row (NocoDB)  
  - Create a row2 (NocoDB)  
  - Sticky Note4, Sticky Note7  

- **Node Details:**

  - **Loop Over Items1**  
    - *Type:* Split In Batches  
    - *Role:* Processes commenters one at a time to avoid rate limits and allow delays  
    - *Connections:* Two outputs: one to `Merge1` for pagination, one to `Wait` before sending DM  

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Adds a randomized delay (between 8 and 11 seconds) before sending each DM to simulate natural behavior  
    - *Connections:* To `Send DM with Lead Magnet`  
    - *Note:* Webhook with random wait time to reduce detection risk  

  - **Send DM with Lead Magnet**  
    - *Type:* HTTP Request  
    - *Role:* Sends DM via Unipile API to commenter’s LinkedIn ID with a personalized message including the lead magnet link and first name  
    - *Batching:* Sends one message per batch with random interval between 8-12 seconds (randomized)  
    - *OnError:* Continues workflow even if sending fails (e.g. LinkedIn API issues)  
    - *Connections:* Outputs to `Create Message Rotation`  

  - **Create Message Rotation**  
    - *Type:* Code  
    - *Role:* Generates a variable reply message from a predefined set to comment under the original comment, increasing naturalness  
    - *Logic:* Cycles through a list of friendly confirmation messages based on run index  
    - *Connections:* Outputs `reply_message` for `Answer to comment`  

  - **Answer to comment**  
    - *Type:* HTTP Request  
    - *Role:* Posts a comment reply on the commenter’s original comment with the rotated message  
    - *OnError:* Continues on failure  
    - *Connections:* Outputs to `Create a row`  

  - **Ask For connection**  
    - *Type:* HTTP Request  
    - *Role:* Posts a comment asking the commenter to send a connection request first if not already connected  
    - *Batching:* Sends one comment every 3 seconds to avoid spamming  
    - *OnError:* Continues on failure  
    - *Connections:* Outputs to `Create a row2`  

  - **Create a row**  
    - *Type:* NocoDB (Create)  
    - *Role:* Logs DM sent status with LinkedIn ID, name, headline, date, post ID, profile URL, connection status, and DM status = "sent"  
    - *Connections:* Loops back to batch processing  

  - **Create a row2**  
    - *Type:* NocoDB (Create)  
    - *Role:* Logs connection request sent status similarly with DM status = "connection request"  

  - *Sticky Note4:* Explains asking for connection by commenting  
  - *Sticky Note7:* Detailed explanation of sending DMs with random wait, personalized messages, and logging  

---

#### 2.5 Pagination Handling & Looping

- **Overview:** Handles multiple pages of comments using cursor-based pagination, merges branches, and loops fetching until no more pages remain.
- **Nodes Involved:**  
  - Merge1 (Merge)  
  - Filter1 (Filter)  
  - cursor (Code)  
  - Sticky Note5, Sticky Note6  

- **Node Details:**

  - **Merge1**  
    - *Type:* Merge  
    - *Role:* Merges two branches of processing: one from DM sending and one from connection requests  
    - *Mode:* Choose branch to continue processing either path  
    - *Connections:* Leads to `Filter1`  

  - **Filter1**  
    - *Type:* Filter  
    - *Role:* Checks if the cursor exists (i.e., if there are more pages of comments to fetch)  
    - *Connections:* True branch loops back to `cursor` node to fetch next page, else ends workflow  

  - **cursor (Code)**  
    - *Role:* Re-initializes cursor for next page fetch, prevents errors if cursor missing  

  - *Sticky Note5:* Notes merging branches and filtering for pagination  
  - *Sticky Note6:* Notes cursor initialization to avoid errors  

---

### 3. Summary Table

| Node Name                               | Node Type             | Functional Role                              | Input Node(s)                              | Output Node(s)                                  | Sticky Note                                                                 |
|----------------------------------------|-----------------------|----------------------------------------------|--------------------------------------------|------------------------------------------------|----------------------------------------------------------------------------|
| On form submission                     | Form Trigger          | Entry point, captures form input              | -                                          | cursor                                         | ## Settings \n- post ID\n- trigger\n- lead magnet URL                      |
| cursor                                | Code                  | Initialize pagination cursor                   | On form submission                         | Get Comments from Posts                         | ## Initialize the cursor to avoid errors                                   |
| Get Comments from Posts                | HTTP Request          | Fetch comments from LinkedIn post via Unipile | cursor                                     | Split Out1                                     | ## Get all the comments from the post \n- get all comments\n- Filter the ones that contains the trigger word |
| Split Out1                           | Split Out             | Extracts comment items array                    | Get Comments from Posts                     | Filter Comments with trigger                    |                                                                            |
| Filter Comments with trigger           | Filter                | Filter comments containing trigger word       | Split Out1                                 | Check if we are connected                       |                                                                            |
| Check if we are connected              | If                    | Check if commenter is 1st degree connection   | Filter Comments with trigger                | Get all the rows where the dm is already sent, Get all the rows where the connection is already asked | ## Check if we are connected\n- check the connection level\n- if it's 1 we can send the DM directly\n- if not we first ask them to connect |
| Get all the rows where the dm is already sent | NocoDB (Get All)     | Fetch records with DM sent status              | Check if we are connected                   | Filter people where we have not sent the DM yet | ## Filter the ones where already have sent messages\nIn both cases, check Nocodb to see if we already have sent a DM or a connection request to avoid double-sends. |
| Filter people where we have not sent the DM yet | Code                  | Filter out commenters who already got DM      | Get all the rows where the dm is already sent, Check if we are connected | If1                                            |                                                                            |
| If1                                   | If                    | Check if filtered DM list is non-empty         | Filter people where we have not sent the DM yet | Split Out2                                    |                                                                            |
| Split Out2                           | Split Out             | Split filtered DM list into individual items   | If1                                         | Loop Over Items1                                |                                                                            |
| Loop Over Items1                     | Split In Batches      | Process commenters one by one                   | Split Out2                                  | Merge1, Wait                                   |                                                                            |
| Wait                                  | Wait                  | Random delay before sending DM                   | Loop Over Items1                            | Send DM with Lead Magnet                        | ## Send the DMs\n- Create a loop so we can log and send them one by one. \n- Add a wait time of a few minutes to avoid problems with linkedin ates and make it random to make it seems more natural. \n- Send the DM including the first name and the link to the lead magnet. \n- Comment a comment with a variable message to make it seem more natural. \n- Log everything into Nocodb. |
| Send DM with Lead Magnet               | HTTP Request          | Send personalized DM including lead magnet link | Wait                                        | Create Message Rotation                         |                                                                            |
| Create Message Rotation                | Code                  | Generate varied reply messages for comments    | Send DM with Lead Magnet                     | Answer to comment                              |                                                                            |
| Answer to comment                     | HTTP Request          | Reply to comment with rotated message          | Create Message Rotation                      | Create a row                                   |                                                                            |
| Create a row                         | NocoDB (Create)       | Log DM sent record                              | Answer to comment                           | Loop Over Items1                               |                                                                            |
| Get all the rows where the connection is already asked | NocoDB (Get All)      | Fetch records with connection request status   | Check if we are connected                   | Filter people where we have not asked connexion yet |                                                                            |
| Filter people where we have not asked connexion yet | Code                  | Filter commenters who haven't been asked connection | Get all the rows where the connection is already asked, Check if we are connected | If                                             |                                                                            |
| If                                    | If                    | Check if filtered connection request list is non-empty | Filter people where we have not asked connexion yet | Split Out                                      |                                                                            |
| Split Out                            | Split Out             | Split filtered connection request list          | If                                           | Ask For connection                             |                                                                            |
| Ask For connection                   | HTTP Request          | Comment asking for connection request            | Split Out                                    | Create a row2                                  | ## Ask them to connect\n- Answer to their comment by asking them to connect. |
| Create a row2                       | NocoDB (Create)       | Log connection request sent record               | Ask For connection                          | Merge1                                         |                                                                            |
| Merge1                              | Merge                 | Merge DM and connection request branches         | Loop Over Items1, Create a row2              | Filter1                                        | ## Merge both branches and filter if there is more pages                   |
| Filter1                            | Filter                | Check if cursor exists (more pages)                | Merge1                                       | cursor                                         |                                                                            |
| Sticky Note                        | Sticky Note           | Settings description                               | -                                            | -                                              | ## Settings \n- post ID\n- trigger\n- lead magnet URL                      |
| Sticky Note1                      | Sticky Note           | Comment retrieval and filtering description        | -                                            | -                                              | ## Get all the comments from the post \n- get all comments\n- Filter the ones that contains the trigger word |
| Sticky Note2                      | Sticky Note           | Connection check description                        | -                                            | -                                              | ## Check if we are connected\n- check the connection level\n- if it's 1 we can send the DM directly\n- if not we first ask them to connect |
| Sticky Note3                      | Sticky Note           | Filtering sent messages and connection requests    | -                                            | -                                              | ## Filter the ones where already have sent messages\nIn both cases, check Nocodb to see if we already have sent a DM or a connection request to avoid double-sends. |
| Sticky Note4                      | Sticky Note           | Asking connection request comment description       | -                                            | -                                              | ## Ask them to connect\n- Answer to their comment by asking them to connect. |
| Sticky Note5                      | Sticky Note           | Merge and pagination description                      | -                                            | -                                              | ## Merge both branches and filter if there is more pages                   |
| Sticky Note6                      | Sticky Note           | Cursor initialization description                      | -                                            | -                                              | ## Initialize the cursor to avoid errors                                   |
| Sticky Note7                      | Sticky Note           | DM sending and logging explanation                      | -                                            | -                                              | ## Send the DMs\n- Create a loop so we can log and send them one by one. \n- Add a wait time of a few minutes to avoid problems with linkedin ates and make it random to make it seems more natural. \n- Send the DM including the first name and the link to the lead magnet. \n- Comment a comment with a variable message to make it seem more natural. \n- Log everything into Nocodb. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node:**  
   - Type: Form Trigger  
   - Name: `On form submission`  
   - Configure webhook with fields:  
     - Post ID (required)  
     - Trigger word  
     - Lead Magnet Link  

2. **Add a Code node:**  
   - Name: `cursor`  
   - Purpose: Initialize cursor variable safely  
   - Code (JavaScript):  
     ```js
     let cursor = "";
     try {
       cursor = $('Get Comments from Posts').first().json.cursor;
     } catch (e) {
       cursor = "";
     }
     return { cursor };
     ```
   - Connect `On form submission` output to `cursor` input.  

3. **Add HTTP Request node to fetch comments:**  
   - Name: `Get Comments from Posts`  
   - Method: GET  
   - URL: `{{$vars.unipileroot}}/posts/urn:li:activity:{{ $('On form submission').first().json["Post ID"] }}/comments`  
   - Query Parameters:  
     - `account_id`: `{{$vars.unipile_linkedin}}`  
     - `cursor`: `{{$json.cursor}}`  
   - Authentication: HTTP Header Auth with UNIPILE credentials  
   - Connect `cursor` output to this node.  

4. **Add Split Out node:**  
   - Name: `Split Out1`  
   - Field to split out: `items`  
   - Connect `Get Comments from Posts` output to `Split Out1`.  

5. **Add Filter node:**  
   - Name: `Filter Comments with trigger`  
   - Condition: Check if comment text (lowercase) contains trigger word from form:  
     Expression example:  
     `{{$json.text.toLowerCase()}}` contains `{{ $('On form submission').first().json["Trigger word"] }}`  
   - Connect `Split Out1` output to this filter.  

6. **Add If node:**  
   - Name: `Check if we are connected`  
   - Condition: Check if `author_details.network_distance` equals `DISTANCE_1`  
   - Connect `Filter Comments with trigger` output to this node.  

7. **Add two NocoDB Get All nodes:**  
   - Names:  
     - `Get all the rows where the dm is already sent`  
       - Table: your table  
       - Filter: `dm_status = sent`  
     - `Get all the rows where the connection is already asked`  
       - Table: your table  
       - Filter: `dm_status = connection request`  
   - Connect `Check if we are connected` True branch to the DM Get All; False branch to connection request Get All.  

8. **Add two Code nodes to filter commenters:**  
   - `Filter people where we have not sent the DM yet`  
     - Filters connected commenters against DM sent list  
   - `Filter people where we have not asked connexion yet`  
     - Filters non-connected commenters against request list, excluding those with reply_counter > 0  
   - Connect each NocoDB node to its respective filter code node.  

9. **Add two If nodes to check non-empty filtered lists:**  
   - `If1` for DM filtered list  
   - `If` for connection request filtered list  
   - Connect filtered code nodes to these If nodes.  

10. **Add two Split Out nodes:**  
    - `Split Out2` for DM filtered list  
    - `Split Out` for connection request filtered list  
    - Connect If nodes True outputs to these Split Outs.  

11. **For DM path:**  
    - Add `Loop Over Items1` (Split In Batches) node, batch size 1  
    - Connect `Split Out2` to `Loop Over Items1`  
    - Add a `Wait` node with randomized delay (e.g., 8-11 seconds) connected to `Loop Over Items1`  
    - Add HTTP Request `Send DM with Lead Magnet` node:  
      - POST to `{{$vars.unipileroot}}/chats`  
      - Body includes account_id, attendees_ids (commenter ID), and a personalized text with first name and lead magnet link from form  
      - Auth: UNIPILE credentials  
      - Enable batching with randomized batchInterval (8000-12000 ms)  
    - Connect `Wait` to `Send DM with Lead Magnet`  

12. **Add Code node `Create Message Rotation`:**  
    - Contains array of varied confirmation messages  
    - Outputs `reply_message` based on run index modulo messages length  
    - Connect `Send DM with Lead Magnet` to this node  

13. **Add HTTP Request node `Answer to comment`:**  
    - POST to `{{$vars.unipileroot}}/posts/{{comment post urn}}/comments`  
    - Body includes account_id, comment_id, and text (from `reply_message`)  
    - Auth: UNIPILE credentials  
    - Connect `Create Message Rotation` to this node  

14. **Add NocoDB create row `Create a row`:**  
    - Logs DM sent with relevant fields (linkedin_id, headline, name, date, post_id, url, connection_status, dm_status = "sent")  
    - Connect `Answer to comment` to this node  
    - Connect back to `Loop Over Items1` to continue batch processing  

15. **For connection request path:**  
    - Connect `Split Out` to `Ask For connection` HTTP Request node:  
      - POST to comments endpoint with text: "Please, send me a connection request first!"  
      - Auth: UNIPILE credentials  
      - Batching enabled with batchInterval ~3000 ms  
    - Connect `Ask For connection` to NocoDB create row `Create a row2` logging connection request similarly with dm_status = "connection request"  
    - Connect `Create a row2` back to merge node  

16. **Add Merge node `Merge1`:**  
    - Mode: Choose Branch (merges DM and connection request branches)  
    - Connect outputs of `Loop Over Items1` and `Create a row2` to `Merge1`  

17. **Add Filter node `Filter1` to check cursor existence:**  
    - Condition: Check if `cursor` property exists (indicating more pages)  
    - Connect `Merge1` to `Filter1`  

18. **Loop back:**  
    - True branch of `Filter1` connects to `cursor` node to fetch next page of comments  
    - False branch ends workflow  

19. **Add Sticky Notes at appropriate places** for documentation and clarity (optional but recommended):  
    - Settings input description near form trigger  
    - Comment retrieval explanation near comment fetch nodes  
    - Connection check explanation near connection check nodes  
    - Messaging explanation near DM sending nodes  
    - Pagination explanation near merge and filter nodes  

20. **Credentials setup:**  
    - UNIPILE HTTP Header Auth credential with required API keys  
    - NocoDB API Token credential with access to the project and table  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow uses Unipile API for LinkedIn data integration, requiring valid Unipile API credentials.         | UNIPILE API: https://unipile.com/api (access and authentication details)                             |
| NocoDB serves as a backend database to avoid duplicate messaging and track outreach status per LinkedIn ID.    | NocoDB docs: https://docs.nocodb.com/                                                               |
| Randomized wait times and message rotation are implemented to reduce risk of LinkedIn spam detection.          | Best practices for automation on LinkedIn: https://www.linkedin.com/help/linkedin/answer/             |
| Form Trigger can be adapted or substituted by other webhook or trigger nodes according to user needs.          | n8n Form Trigger docs: https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger/                          |
| Ensure the LinkedIn post ID format matches Unipile API expectations (URN format).                              | LinkedIn URN format info: https://docs.microsoft.com/en-us/linkedin/shared/integrations/people/urns  |

---

**Disclaimer:** The text provided is exclusively generated from an automated n8n workflow designed for lawful and public data handling, compliant with all applicable content policies.