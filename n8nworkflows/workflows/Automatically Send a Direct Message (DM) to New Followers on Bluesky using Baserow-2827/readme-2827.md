Automatically Send a Direct Message (DM) to New Followers on Bluesky using Baserow

https://n8nworkflows.xyz/workflows/automatically-send-a-direct-message--dm--to-new-followers-on-bluesky-using-baserow-2827


# Automatically Send a Direct Message (DM) to New Followers on Bluesky using Baserow

### 1. Workflow Overview

This workflow automates sending personalized direct messages (DMs) to new followers on Bluesky, leveraging a Baserow database to track followers and prevent duplicate messages. It is designed for creators and community managers who want to maintain engagement by welcoming new followers consistently without manual effort.

The workflow runs daily at 9 AM, fetches the latest followers from Bluesky, compares them against a stored database, extracts first names when possible, and sends customized welcome messages with optional links. It includes double verification to avoid sending duplicate messages and updates the database accordingly.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Credential Setup:** Initiates the workflow daily and sets Bluesky credentials.
- **1.2 Bluesky Session & Followers Retrieval:** Creates a session and fetches the latest followers.
- **1.3 Followers Extraction & Looping:** Extracts follower data and iterates through each follower.
- **1.4 Database Check & Record Management:** Checks if followers exist in Baserow, creates new records if needed.
- **1.5 New Followers Processing & Messaging:** Identifies new followers, extracts first names, retrieves conversation IDs, sends personalized welcome messages, and updates records.
- **1.6 Rate Limiting & Workflow Termination:** Implements limits and waits to handle API constraints and ends the workflow gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Credential Setup

- **Overview:** This block triggers the workflow daily at 9 AM and sets the Bluesky handle and app password credentials for API access.
- **Nodes Involved:**  
  - Run Daily at 9 AM  
  - Set Bluesky Credentials  
  - Sticky Note5 (contextual comment)

- **Node Details:**

  - **Run Daily at 9 AM**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every day at 9 AM automatically.  
    - Configuration: Default daily trigger time (9 AM).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Set Bluesky Credentials".  
    - Edge Cases: Workflow won't run if n8n instance is down at trigger time.

  - **Set Bluesky Credentials**  
    - Type: Set  
    - Role: Stores Bluesky handle and app password as workflow variables for authentication.  
    - Configuration: User inputs Bluesky handle and app password here.  
    - Inputs: From schedule trigger.  
    - Outputs: Connects to "Create Bluesky Session".  
    - Edge Cases: Missing or incorrect credentials will cause authentication failures downstream.

  - **Sticky Note5**  
    - Content: "Set your Bluesky handle and app password here to authenticate API requests."  
    - Applies to: "Set Bluesky Credentials" node.

---

#### 1.2 Bluesky Session & Followers Retrieval

- **Overview:** Establishes a session with Bluesky API and retrieves the latest followers list.
- **Nodes Involved:**  
  - Create Bluesky Session  
  - Get Latest Followers  
  - Extract Followers Array  
  - Sticky Note, Sticky Note1 (contextual comments)

- **Node Details:**

  - **Create Bluesky Session**  
    - Type: HTTP Request  
    - Role: Initiates a session with Bluesky API using credentials.  
    - Configuration: Uses Bluesky handle and app password from previous node to authenticate.  
    - Inputs: From "Set Bluesky Credentials".  
    - Outputs: Connects to "Get Latest Followers".  
    - Edge Cases: Authentication errors, network timeouts, API rate limits.

  - **Get Latest Followers**  
    - Type: HTTP Request  
    - Role: Fetches the current list of followers from Bluesky API.  
    - Configuration: Uses session token from "Create Bluesky Session".  
    - Inputs: From "Create Bluesky Session".  
    - Outputs: Connects to "Extract Followers Array".  
    - Edge Cases: API errors, empty follower list, pagination issues if followers > API page size.

  - **Extract Followers Array**  
    - Type: Code  
    - Role: Parses the raw API response to extract an array of follower objects.  
    - Configuration: Custom JavaScript code to extract relevant follower data.  
    - Inputs: From "Get Latest Followers".  
    - Outputs: Connects to "Loop Followers".  
    - Edge Cases: Malformed API response, missing fields.

  - **Sticky Note**  
    - Content: "This block authenticates and fetches your current Bluesky followers."  
    - Applies to: "Create Bluesky Session", "Get Latest Followers".

  - **Sticky Note1**  
    - Content: "Extracts follower data into an array for processing."  
    - Applies to: "Extract Followers Array".

---

#### 1.3 Followers Extraction & Looping

- **Overview:** Iterates over each follower to check their existence in the database and process accordingly.
- **Nodes Involved:**  
  - Loop Followers  
  - Get Follower Record  
  - If Follower Exists  
  - Create Follower Record  
  - Wait Follower Loop  
  - Limit  
  - Sticky Note2 (contextual comment)

- **Node Details:**

  - **Loop Followers**  
    - Type: SplitInBatches  
    - Role: Processes followers in batches to manage API limits and workflow performance.  
    - Configuration: Batch size configured to handle up to 100 followers per run.  
    - Inputs: From "Extract Followers Array".  
    - Outputs: Two outputs: one to "Limit" and one to "Get Follower Record".  
    - Edge Cases: Large follower lists may require batch size tuning.

  - **Get Follower Record**  
    - Type: Baserow  
    - Role: Queries Baserow database to check if follower record exists.  
    - Configuration: Uses follower unique ID or handle as query parameter.  
    - Inputs: From "Loop Followers".  
    - Outputs: Connects to "If Follower Exists".  
    - Edge Cases: Database connection errors, missing records.

  - **If Follower Exists**  
    - Type: If  
    - Role: Branches workflow based on whether follower is already recorded.  
    - Configuration: Checks if query result is empty or not.  
    - Inputs: From "Get Follower Record".  
    - Outputs:  
      - True: Loops back to "Loop Followers" (no new record needed).  
      - False: Proceeds to "Create Follower Record".  
    - Edge Cases: Expression evaluation errors.

  - **Create Follower Record**  
    - Type: Baserow  
    - Role: Adds new follower record to database.  
    - Configuration: Inserts follower details, sets SentWelcome flag to false.  
    - Inputs: From "If Follower Exists" (False branch).  
    - Outputs: Connects to "Wait Follower Loop".  
    - Edge Cases: Database write failures.

  - **Wait Follower Loop**  
    - Type: Wait  
    - Role: Adds delay between follower processing to avoid API rate limits.  
    - Configuration: Default wait time (e.g., 1 second).  
    - Inputs: From "Create Follower Record".  
    - Outputs: Loops back to "Loop Followers".  
    - Edge Cases: Workflow delays, potential timeout if too long.

  - **Limit**  
    - Type: Limit  
    - Role: Limits the number of followers processed per run to avoid overload.  
    - Configuration: Limits to 100 followers per execution.  
    - Inputs: From "Loop Followers".  
    - Outputs: Connects to "Get All New Followers".  
    - Edge Cases: May skip followers if more than limit; requires multiple runs.

  - **Sticky Note2**  
    - Content: "This block checks if followers exist in the database and adds new ones."  
    - Applies to: "Loop Followers", "Get Follower Record", "If Follower Exists", "Create Follower Record".

---

#### 1.4 Database Check & Record Management for New Followers

- **Overview:** Retrieves all new followers (those without a welcome message sent) from the database and processes them for messaging.
- **Nodes Involved:**  
  - Get All New Followers  
  - Loop New Followers  
  - Get Firstname  
  - Get ConvoId  
  - Double Check If Welcome Has Already Been Sent  
  - Create Welcome Message  
  - Send Welcome Message  
  - Update Follower Record to SentWelcome = TRUE  
  - Wait New Follower Loop  
  - END OF WORKFLOW  
  - Sticky Note6 (contextual comment)

- **Node Details:**

  - **Get All New Followers**  
    - Type: Baserow  
    - Role: Queries database for followers with SentWelcome flag = false.  
    - Configuration: Filter on SentWelcome = false.  
    - Inputs: From "Limit".  
    - Outputs: Connects to "Loop New Followers".  
    - Edge Cases: Database query errors, empty result sets.

  - **Loop New Followers**  
    - Type: SplitInBatches  
    - Role: Processes new followers in batches for messaging.  
    - Configuration: Batch size to manage API limits.  
    - Inputs: From "Get All New Followers".  
    - Outputs: Two outputs: one to "END OF WORKFLOW" (empty batch), one to "Get Firstname".  
    - Edge Cases: Batch size tuning needed.

  - **Get Firstname**  
    - Type: Code  
    - Role: Extracts the follower’s first name from profile data if available.  
    - Configuration: Custom JavaScript parsing follower’s display name or profile.  
    - Inputs: From "Loop New Followers".  
    - Outputs: Connects to "Get ConvoId".  
    - Edge Cases: Missing or malformed name data.

  - **Get ConvoId**  
    - Type: HTTP Request  
    - Role: Retrieves conversation ID for sending a DM to the follower.  
    - Configuration: Uses Bluesky API with follower handle.  
    - Inputs: From "Get Firstname".  
    - Outputs: Connects to "Double Check If Welcome Has Already Been Sent".  
    - Edge Cases: API errors, missing conversation ID.

  - **Double Check If Welcome Has Already Been Sent**  
    - Type: If  
    - Role: Verifies again if the welcome message was already sent to avoid duplicates.  
    - Configuration: Checks database SentWelcome flag or message history.  
    - Inputs: From "Get ConvoId".  
    - Outputs:  
      - True: Connects to "Update Follower Record to SentWelcome = TRUE" (skip sending).  
      - False: Connects to "Create Welcome Message" (proceed to send).  
    - Edge Cases: Logic errors causing duplicate messages.

  - **Create Welcome Message**  
    - Type: Code  
    - Role: Generates a personalized welcome message, optionally including a link.  
    - Configuration: Uses follower’s first name and customizable message template.  
    - Inputs: From "Double Check If Welcome Has Already Been Sent" (False branch).  
    - Outputs: Connects to "Send Welcome Message".  
    - Edge Cases: Template errors, missing variables.

  - **Send Welcome Message**  
    - Type: HTTP Request  
    - Role: Sends the DM via Bluesky API using conversation ID and message content.  
    - Configuration: Uses API token, conversation ID, and message body.  
    - Inputs: From "Create Welcome Message".  
    - Outputs: Connects to "Update Follower Record to SentWelcome = TRUE".  
    - Edge Cases: API failures, rate limits, message rejections.

  - **Update Follower Record to SentWelcome = TRUE**  
    - Type: Baserow  
    - Role: Updates the follower record to mark that the welcome message was sent.  
    - Configuration: Sets SentWelcome field to true.  
    - Inputs: From "Send Welcome Message" or "Double Check If Welcome Has Already Been Sent" (True branch).  
    - Outputs: Connects to "Wait New Follower Loop".  
    - Edge Cases: Database write errors.

  - **Wait New Follower Loop**  
    - Type: Wait  
    - Role: Adds delay between sending messages to avoid API rate limits.  
    - Configuration: Default wait time (e.g., 1 second).  
    - Inputs: From "Update Follower Record to SentWelcome = TRUE".  
    - Outputs: No further output (end of loop).  
    - Edge Cases: Workflow delays.

  - **END OF WORKFLOW**  
    - Type: NoOp  
    - Role: Marks the end of the workflow execution.  
    - Inputs: From "Loop New Followers" (empty batch).  
    - Outputs: None.

  - **Sticky Note6**  
    - Content: "This block handles sending personalized welcome messages and updates the database to prevent duplicates."  
    - Applies to: All nodes in this block.

---

#### 1.5 Rate Limiting & Workflow Termination

- **Overview:** Implements rate limiting via wait nodes and limits node to ensure API compliance and graceful workflow termination.
- **Nodes Involved:**  
  - Limit  
  - Wait Follower Loop  
  - Wait New Follower Loop  
  - END OF WORKFLOW

- **Node Details:**

  - **Limit**  
    - See above in 1.3.

  - **Wait Follower Loop**  
    - See above in 1.3.

  - **Wait New Follower Loop**  
    - See above in 1.4.

  - **END OF WORKFLOW**  
    - See above in 1.4.

---

### 3. Summary Table

| Node Name                              | Node Type           | Functional Role                                  | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                      |
|--------------------------------------|---------------------|-------------------------------------------------|----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Run Daily at 9 AM                    | Schedule Trigger    | Triggers workflow daily at 9 AM                  | None                             | Set Bluesky Credentials                |                                                                                                |
| Set Bluesky Credentials              | Set                 | Stores Bluesky handle and app password           | Run Daily at 9 AM                | Create Bluesky Session                 | Set your Bluesky handle and app password here to authenticate API requests.                    |
| Create Bluesky Session               | HTTP Request        | Authenticates and creates Bluesky API session    | Set Bluesky Credentials          | Get Latest Followers                   | This block authenticates and fetches your current Bluesky followers.                           |
| Get Latest Followers                 | HTTP Request        | Fetches latest followers from Bluesky             | Create Bluesky Session           | Extract Followers Array                | This block authenticates and fetches your current Bluesky followers.                           |
| Extract Followers Array              | Code                | Parses API response to extract followers array    | Get Latest Followers             | Loop Followers                        | Extracts follower data into an array for processing.                                          |
| Loop Followers                      | SplitInBatches      | Iterates over followers in batches                 | Extract Followers Array          | Limit, Get Follower Record             | This block checks if followers exist in the database and adds new ones.                       |
| Get Follower Record                 | Baserow             | Checks if follower exists in database              | Loop Followers                  | If Follower Exists                    | This block checks if followers exist in the database and adds new ones.                       |
| If Follower Exists                  | If                  | Branches based on follower existence               | Get Follower Record             | Loop Followers (True), Create Follower Record (False) | This block checks if followers exist in the database and adds new ones.                       |
| Create Follower Record              | Baserow             | Adds new follower record to database                | If Follower Exists (False)      | Wait Follower Loop                    | This block checks if followers exist in the database and adds new ones.                       |
| Wait Follower Loop                  | Wait                | Adds delay between follower processing              | Create Follower Record          | Loop Followers                       | This block checks if followers exist in the database and adds new ones.                       |
| Limit                             | Limit               | Limits number of followers processed per run        | Loop Followers                  | Get All New Followers                |                                                                                                |
| Get All New Followers              | Baserow             | Retrieves followers without welcome message sent    | Limit                          | Loop New Followers                   |                                                                                                |
| Loop New Followers                | SplitInBatches      | Processes new followers in batches for messaging    | Get All New Followers           | END OF WORKFLOW (empty batch), Get Firstname | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Get Firstname                    | Code                | Extracts first name from follower profile            | Loop New Followers              | Get ConvoId                        | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Get ConvoId                     | HTTP Request        | Retrieves conversation ID for DM                      | Get Firstname                  | Double Check If Welcome Has Already Been Sent | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Double Check If Welcome Has Already Been Sent | If                  | Verifies if welcome message was already sent         | Get ConvoId                   | Update Follower Record to SentWelcome=TRUE (True), Create Welcome Message (False) | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Create Welcome Message           | Code                | Generates personalized welcome message                | Double Check If Welcome Has Already Been Sent (False) | Send Welcome Message             | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Send Welcome Message            | HTTP Request        | Sends DM via Bluesky API                               | Create Welcome Message         | Update Follower Record to SentWelcome=TRUE | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Update Follower Record to SentWelcome = TRUE | Baserow             | Updates database to mark welcome message sent         | Send Welcome Message, Double Check If Welcome Has Already Been Sent (True) | Wait New Follower Loop            | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| Wait New Follower Loop          | Wait                | Adds delay between sending messages                    | Update Follower Record to SentWelcome=TRUE | None                            | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |
| END OF WORKFLOW                | NoOp                | Marks end of workflow execution                         | Loop New Followers (empty batch) | None                            |                                                                                                |
| Sticky Note                    | Sticky Note         | Comment on Bluesky session and followers retrieval    | None                          | None                            | This block authenticates and fetches your current Bluesky followers.                           |
| Sticky Note1                   | Sticky Note         | Comment on followers extraction                         | None                          | None                            | Extracts follower data into an array for processing.                                          |
| Sticky Note2                   | Sticky Note         | Comment on database check and record creation          | None                          | None                            | This block checks if followers exist in the database and adds new ones.                       |
| Sticky Note5                   | Sticky Note         | Comment on setting Bluesky credentials                  | None                          | None                            | Set your Bluesky handle and app password here to authenticate API requests.                    |
| Sticky Note6                   | Sticky Note         | Comment on sending welcome messages and updating DB    | None                          | None                            | This block handles sending personalized welcome messages and updates the database to prevent duplicates. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "Automatically Send a Direct Message (DM) to New Followers on Bluesky".**

2. **Add a Schedule Trigger node:**
   - Name: "Run Daily at 9 AM"
   - Set to trigger daily at 9:00 AM.

3. **Add a Set node:**
   - Name: "Set Bluesky Credentials"
   - Add two fields:  
     - `blueskyHandle` (string): Your Bluesky username/handle.  
     - `appPassword` (string): Your Bluesky app password.
   - Connect "Run Daily at 9 AM" → "Set Bluesky Credentials".

4. **Add an HTTP Request node:**
   - Name: "Create Bluesky Session"
   - Configure to authenticate with Bluesky API using credentials from "Set Bluesky Credentials".  
   - Use appropriate API endpoint to create a session or authenticate.
   - Connect "Set Bluesky Credentials" → "Create Bluesky Session".

5. **Add an HTTP Request node:**
   - Name: "Get Latest Followers"
   - Configure to call Bluesky API endpoint to fetch followers list, using session token from "Create Bluesky Session".
   - Connect "Create Bluesky Session" → "Get Latest Followers".

6. **Add a Code node:**
   - Name: "Extract Followers Array"
   - Write JavaScript code to parse the API response and extract an array of follower objects.
   - Connect "Get Latest Followers" → "Extract Followers Array".

7. **Add a SplitInBatches node:**
   - Name: "Loop Followers"
   - Set batch size (e.g., 100).
   - Connect "Extract Followers Array" → "Loop Followers".

8. **Add a Baserow node:**
   - Name: "Get Follower Record"
   - Configure to query your Baserow database for follower record by unique identifier (e.g., handle or ID).
   - Connect "Loop Followers" → "Get Follower Record".

9. **Add an If node:**
   - Name: "If Follower Exists"
   - Condition: Check if "Get Follower Record" returned any records.
   - True branch: Connect back to "Loop Followers" to skip existing followers.
   - False branch: Connect to "Create Follower Record".

10. **Add a Baserow node:**
    - Name: "Create Follower Record"
    - Configure to insert new follower record with details and set `SentWelcome` flag to false.
    - Connect "If Follower Exists" (False) → "Create Follower Record".

11. **Add a Wait node:**
    - Name: "Wait Follower Loop"
    - Set wait time (e.g., 1 second) to avoid API rate limits.
    - Connect "Create Follower Record" → "Wait Follower Loop".
    - Connect "Wait Follower Loop" → "Loop Followers" (to continue processing).

12. **Add a Limit node:**
    - Name: "Limit"
    - Set maximum items to process per run (e.g., 100).
    - Connect "Loop Followers" → "Limit".

13. **Add a Baserow node:**
    - Name: "Get All New Followers"
    - Configure to query Baserow for followers where `SentWelcome` = false.
    - Connect "Limit" → "Get All New Followers".

14. **Add a SplitInBatches node:**
    - Name: "Loop New Followers"
    - Set batch size (e.g., 100).
    - Connect "Get All New Followers" → "Loop New Followers".

15. **Add a Code node:**
    - Name: "Get Firstname"
    - Write JavaScript to extract first name from follower profile data.
    - Connect "Loop New Followers" → "Get Firstname".

16. **Add an HTTP Request node:**
    - Name: "Get ConvoId"
    - Configure to call Bluesky API to get conversation ID for DM with follower.
    - Use follower handle from previous node.
    - Connect "Get Firstname" → "Get ConvoId".

17. **Add an If node:**
    - Name: "Double Check If Welcome Has Already Been Sent"
    - Condition: Verify if welcome message was already sent (check `SentWelcome` flag or message history).
    - True branch: Connect to "Update Follower Record to SentWelcome = TRUE".
    - False branch: Connect to "Create Welcome Message".

18. **Add a Code node:**
    - Name: "Create Welcome Message"
    - Write JavaScript to generate personalized welcome message using first name and optional links.
    - Connect "Double Check If Welcome Has Already Been Sent" (False) → "Create Welcome Message".

19. **Add an HTTP Request node:**
    - Name: "Send Welcome Message"
    - Configure to send DM via Bluesky API using conversation ID and message content.
    - Connect "Create Welcome Message" → "Send Welcome Message".

20. **Add a Baserow node:**
    - Name: "Update Follower Record to SentWelcome = TRUE"
    - Configure to update follower record in Baserow, setting `SentWelcome` to true.
    - Connect "Send Welcome Message" → "Update Follower Record to SentWelcome = TRUE".
    - Also connect "Double Check If Welcome Has Already Been Sent" (True) → "Update Follower Record to SentWelcome = TRUE".

21. **Add a Wait node:**
    - Name: "Wait New Follower Loop"
    - Set wait time (e.g., 1 second) to avoid API rate limits.
    - Connect "Update Follower Record to SentWelcome = TRUE" → "Wait New Follower Loop".

22. **Add a NoOp node:**
    - Name: "END OF WORKFLOW"
    - Connect "Loop New Followers" (empty batch output) → "END OF WORKFLOW".

---

**Credentials Setup:**

- Configure Bluesky API credentials in "Set Bluesky Credentials" node.
- Configure Baserow API credentials in all Baserow nodes.
- Ensure HTTP Request nodes use proper authentication headers or tokens as required by Bluesky API.

---

This completes the full reproduction of the workflow, enabling automated, personalized welcome messaging to new Bluesky followers with database tracking and rate limiting safeguards.