Send a welcome private message to your new BlueSky followers

https://n8nworkflows.xyz/workflows/send-a-welcome-private-message-to-your-new-bluesky-followers-2570


# Send a welcome private message to your new BlueSky followers

### 1. Workflow Overview

This workflow automates sending a personalized welcome private message to new followers on BlueSky. It targets BlueSky users who want to acknowledge new followers via private messages automatically.

The workflow runs every 60 minutes, checking for new followers by comparing the current follower list with a stored snapshot file. For each new follower, it initiates a private conversation (if not existing) and sends a predefined welcome message with a customizable link.

The workflow is logically divided into these blocks:

- **1.1 Authentication & Session Management:** Creates a session with BlueSky API using user credentials.
- **1.2 Fetching Current Followers:** Retrieves the current list of followers from BlueSky.
- **1.3 Reading Stored Followers:** Reads the saved local file containing previously known followers.
- **1.4 Identifying New Followers:** Compares current followers with stored ones to isolate new followers.
- **1.5 Message Preparation:** Defines the welcome message text and associated link.
- **1.6 Messaging New Followers:** For each new follower, gets or creates a conversation and sends the welcome message.
- **1.7 Updating Stored Followers:** Saves the updated follower list after messages are sent.

---

### 2. Block-by-Block Analysis

#### 1.1 Authentication & Session Management

- **Overview:** Authenticates the user and generates an access session token for subsequent API calls.
- **Nodes Involved:** `Each 60 minutes`, `Create Session`
- **Node Details:**

  - **Each 60 minutes**
    - Type: Schedule Trigger
    - Role: Triggers the workflow every 60 minutes.
    - Config: Interval set to 60 minutes.
    - Input: None (time-based).
    - Output: Triggers `Create Session`.
    - Edge cases: Workflow will not trigger if n8n is offline; no error expected.

  - **Create Session**
    - Type: HTTP Request (POST)
    - Role: Calls BlueSky API to create a session.
    - Config: 
      - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`
      - Body: Sends user identifier (BlueSky handle) and app password.
      - Output: Contains `accessJwt` token and user info.
    - Parameters to set: User handle and BlueSky app password.
    - Input: Triggered by schedule.
    - Output: Triggers `List followers`.
    - Edge cases: Authentication errors (wrong credentials or expired app password), network timeouts.
    - Notes: Requires user to manually set credentials before enabling workflow.

#### 1.2 Fetching Current Followers

- **Overview:** Retrieves the current list of followers from BlueSky using the authenticated session.
- **Nodes Involved:** `List followers`
- **Node Details:**

  - **List followers**
    - Type: HTTP Request (GET)
    - Role: Fetches followers for the authenticated user.
    - Config:
      - URL: `https://bsky.social/xrpc/app.bsky.graph.getFollowers`
      - Query: Uses the user's DID from `Create Session`.
      - Pagination: Supports up to 2 pages with 100 followers per page, 250ms interval between requests.
      - Headers: Authorization with Bearer token from session.
    - Input: Output from `Create Session`.
    - Output: Followers list JSON.
    - Edge cases: API rate limits, pagination errors, authorization errors.
    - Version: Uses version 4.2 of HTTP Request node.

#### 1.3 Reading Stored Followers

- **Overview:** Reads the previously saved followers from a local JSON file to enable comparison.
- **Nodes Involved:** `Read followers from file`, `Extract from File`
- **Node Details:**

  - **Read followers from file**
    - Type: Read/Write File
    - Role: Reads the JSON file containing saved followers.
    - Config:
      - Filename: `followers-{{Create Session.handle}}.json` (dynamic based on user).
    - Input: Output from `List followers`.
    - Output: File content (JSON string).
    - Edge cases: File not found on first run; must be handled manually initially.

  - **Extract from File**
    - Type: Extract From File
    - Role: Converts JSON string from file into usable JSON data.
    - Config: Operation set to "fromJson".
    - Input: Output from `Read followers from file`.
    - Output: Parsed JSON.
    - Edge cases: Malformed JSON file, empty file.

#### 1.4 Identifying New Followers

- **Overview:** Compares the current followers with the stored list to find new followers.
- **Nodes Involved:** `Find new followers`, `Split Out`
- **Node Details:**

  - **Find new followers**
    - Type: Code (JavaScript)
    - Role: Compares two follower lists and returns new followers only.
    - Code Highlights:
      - Reads followers from `List followers` and `Extract from File`.
      - Creates a Set of existing follower DIDs.
      - Filters current followers to exclude existing ones.
      - Outputs new followers, their DIDs, and count.
    - Input: Outputs from `List followers` and `Extract from File`.
    - Output: New followers data and DIDs.
    - Edge cases: Empty follower lists, missing data, code execution errors.

  - **Split Out**
    - Type: Split Out
    - Role: Extracts array of new follower DIDs into individual items for iteration.
    - Config: Field to split out is `newDids`, destination field name is `did`.
    - Input: Output from `Find new followers`.
    - Output: Individual DID items.
    - Edge cases: Empty arrays result in no output items.

#### 1.5 Message Preparation

- **Overview:** Defines the welcome message text and link to send to new followers.
- **Nodes Involved:** `Define welcome message`, `Sticky Note2`
- **Node Details:**

  - **Define welcome message**
    - Type: Set
    - Role: Sets static message text and link to be used in the welcome message.
    - Config:
      - `text`: "Hello, thanks for your follow. You can read more about my over my site:"
      - `link`: "https://yoursite.com"
    - Input: Optional second output from `Loop Over Items` (triggered only if new followers exist).
    - Output: Message fields for subsequent nodes.
    - Edge cases: None expected unless variables are changed incorrectly.

  - **Sticky Note2**
    - Type: Sticky Note
    - Role: Instruction to customize the welcome message and link.
    - Content: "### 2. Define your welcome message and link here"

#### 1.6 Messaging New Followers

- **Overview:** For each new follower DID, retrieves or creates a conversation and sends the welcome message.
- **Nodes Involved:** `Loop Over Items`, `Get conversation ID`, `Send message`
- **Node Details:**

  - **Loop Over Items**
    - Type: Split In Batches
    - Role: Processes new follower DIDs one at a time.
    - Config: Default batch size (1).
    - Input: Items from `Split Out`.
    - Outputs:
      - Main output (empty): No direct output.
      - Secondary output: Triggers `Define welcome message` only if items exist.
    - Edge cases: No new followers results in no output.

  - **Get conversation ID**
    - Type: HTTP Request (GET)
    - Role: Checks if a conversation exists with the follower or creates one.
    - Config:
      - URL: Uses dynamic service endpoint from `Create Session` data.
      - Query: `members` parameter set to current follower DID.
      - Headers: Authorization and Atproto-Proxy header.
    - Input: From `Define welcome message`.
    - Output: Conversation ID JSON.
    - Edge cases: API errors, missing conversation, network issues.

  - **Send message**
    - Type: HTTP Request (POST)
    - Role: Sends the welcome message to the conversation.
    - Config:
      - URL: Uses service endpoint from `Create Session`.
      - Body: JSON containing conversation ID and message text with a clickable link facet.
      - Headers: Authorization and Atproto-Proxy header.
    - Input: From `Get conversation ID`.
    - Output: None (ends chain).
    - Edge cases: Message sending failure, rate limits.

#### 1.7 Updating Stored Followers

- **Overview:** Saves the updated follower list including new followers to local storage for next comparison.
- **Nodes Involved:** `Convert to File`, `Save followers to file`, `No Operation, do nothing`, `Wait`, `Sticky Note3`
- **Node Details:**

  - **Convert to File**
    - Type: Convert To File
    - Role: Converts current followers JSON into a JSON file format.
    - Config:
      - Filename: `followers-basuracero.json` (example filename).
      - Operation: toJson.
    - Input: From `Wait`.
    - Output: File data.
    - Edge cases: File write errors.

  - **Save followers to file**
    - Type: Read/Write File
    - Role: Writes the current followers file, overwriting existing.
    - Config:
      - Filename: `followers-{{Create Session.handle}}.json`
      - Append: false (overwrite).
    - Input: From `Convert to File`.
    - Output: Final step before no-op.
    - Edge cases: File write permission issues.

  - **No Operation, do nothing**
    - Type: NoOp
    - Role: Placeholder to terminate workflow cleanly.
    - Input: From `Save followers to file`.
    - Output: None.

  - **Wait**
    - Type: Wait
    - Role: Delays execution before saving followers file to ensure all API calls finish.
    - Input: From `List followers`.
    - Output: Triggers `Convert to File`.
    - Edge cases: None.

  - **Sticky Note3**
    - Type: Sticky Note
    - Role: Instruction emphasizing the need to run `Save followers to file` manually once initially.
    - Content: "### 3. **Important** \n\nYou need to manually run \"Save followers to file\" once before the first time so you populate your list of existing followers"

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                     | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                                             |
|------------------------|----------------------|-----------------------------------|--------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Each 60 minutes        | Schedule Trigger     | Triggers workflow every 60 minutes| None                     | Create Session              |                                                                                                                         |
| Create Session         | HTTP Request         | Authenticate and create session   | Each 60 minutes          | List followers              | "### 1. Define your Bluesky user and app password first\n\nThe App password should have access to private messages"       |
| List followers         | HTTP Request         | Fetch current followers           | Create Session           | Read followers from file, Wait |                                                                                                                         |
| Read followers from file| Read/Write File      | Read stored followers JSON file   | List followers           | Extract from File           |                                                                                                                         |
| Extract from File      | Extract From File    | Parse followers JSON              | Read followers from file | Find new followers          |                                                                                                                         |
| Find new followers     | Code                 | Identify new followers by comparison| Extract from File, List followers | Split Out            |                                                                                                                         |
| Split Out              | Split Out            | Split new follower DIDs for loop  | Find new followers       | Loop Over Items             |                                                                                                                         |
| Loop Over Items        | Split In Batches     | Iterate over new followers        | Split Out                | (empty), Define welcome message |                                                                                                                         |
| Define welcome message | Set                  | Define welcome message text & link| Loop Over Items          | Get conversation ID         | "### 2. Define your welcome message and link here"                                                                      |
| Get conversation ID    | HTTP Request         | Get or create conversation ID     | Define welcome message   | Send message                |                                                                                                                         |
| Send message           | HTTP Request         | Send welcome private message      | Get conversation ID      | Loop Over Items (secondary) |                                                                                                                         |
| Wait                   | Wait                 | Delay before saving followers     | List followers           | Convert to File             |                                                                                                                         |
| Convert to File        | Convert To File      | Convert followers to JSON file    | Wait                     | Save followers to file      |                                                                                                                         |
| Save followers to file | Read/Write File      | Save updated followers list       | Convert to File          | No Operation, do nothing    | "### 3. **Important** \n\nYou need to manually run \"Save followers to file\" once before the first time so you populate your list of existing followers" |
| No Operation, do nothing| NoOp                 | Terminate workflow cleanly        | Save followers to file   | None                       |                                                                                                                         |
| Sticky Note            | Sticky Note          | Overview and instructions         | None                     | None                       | "## Send a welcome private message to your new BlueSky followers\n\nThis flow will save your current followers in a file and check for new ones on the next execution, sending them the Defined message an link as a private message.\n\nOnce messages are sent, the new list of followers will be saved into the file.\n\n**Important: Follow the yellow notes in order before enabling the full flow for the first time**" |
| Sticky Note1           | Sticky Note          | Instructions to define credentials| None                     | None                       | "### 1. Define your Bluesky user and app password first\n\nThe App password should have access to private messages"       |
| Sticky Note2           | Sticky Note          | Instructions to customize message | None                     | None                       | "### 2. Define your welcome message and link here"                                                                      |
| Sticky Note3           | Sticky Note          | Reminder to manually run save step| None                     | None                       | "### 3. **Important** \n\nYou need to manually run \"Save followers to file\" once before the first time so you populate your list of existing followers" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Name: `Each 60 minutes`
   - Type: Schedule Trigger
   - Configuration: Set interval to 60 minutes (every 60 minutes).

2. **Create an HTTP Request node for session creation**
   - Name: `Create Session`
   - Type: HTTP Request
   - Method: POST
   - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`
   - Body Parameters: JSON with fields
     - `identifier`: Your BlueSky username (e.g. `youruser.bsky.social`)
     - `password`: Your BlueSky app password (generated from BlueSky settings with private message permission)
   - Connect output of `Each 60 minutes` to input of this node.

3. **Create an HTTP Request node to list followers**
   - Name: `List followers`
   - Type: HTTP Request
   - Method: GET
   - URL: `https://bsky.social/xrpc/app.bsky.graph.getFollowers`
   - Query Parameters:
     - `actor`: Set dynamically to `={{ $json.did }}` from `Create Session` output.
     - `limit`: 100
   - Headers:
     - `Authorization`: `=Bearer {{ $item("0").$node["Create Session"].json["accessJwt"] }}`
   - Pagination:
     - Enable pagination with max 2 pages, 250ms interval.
   - Connect output of `Create Session` to input of this node.

4. **Create Read/Write File node to read followers from JSON file**
   - Name: `Read followers from file`
   - Type: Read/Write File
   - Operation: Read
   - File Selector: `followers-{{ $('Create Session').item.json.handle }}.json`
   - Connect output of `List followers` to input.

5. **Create Extract From File node**
   - Name: `Extract from File`
   - Type: Extract From File
   - Operation: fromJson
   - Connect output of `Read followers from file`.

6. **Create a Code node to find new followers**
   - Name: `Find new followers`
   - Type: Code (JavaScript)
   - Code:
     ```js
     const listFollowers = $('List followers').all()[0].json.followers;
     const extractFromFile = $('Extract from File').all()[0].json.data[0].followers;

     const existingDids = new Set(extractFromFile?.map(item => item.did) || []);
     const newFollowers = listFollowers?.filter(follower => !existingDids.has(follower.did)) || [];

     return {
       json: {
         newFollowers,
         newDids: newFollowers.map(follower => follower.did),
         count: newFollowers.length
       }
     }
     ```
   - Connect output of `Extract from File` and `List followers` to this node.

7. **Create a Split Out node**
   - Name: `Split Out`
   - Type: Split Out
   - Field to split out: `newDids`
   - Destination field name: `did`
   - Connect output of `Find new followers`.

8. **Create a Split In Batches node**
   - Name: `Loop Over Items`
   - Type: Split In Batches
   - Connect output of `Split Out` to input.
   - Configure to process one item at a time.

9. **Create a Set node for defining the welcome message**
   - Name: `Define welcome message`
   - Type: Set
   - Fields:
     - `text`: "Hello, thanks for your follow. You can read more about my over my site:"
     - `link`: "https://yoursite.com"
   - Connect second output of `Loop Over Items` to input of this node (this output triggers only if items exist).

10. **Create an HTTP Request node to get conversation ID**
    - Name: `Get conversation ID`
    - Type: HTTP Request
    - Method: GET
    - URL: `={{ $item("0").$node["Create Session"].json.didDoc.service[0].serviceEndpoint }}/xrpc/chat.bsky.convo.getConvoForMembers`
    - Query Parameter:
      - `members`: `={{ $('Split Out').item.json.did }}`
    - Headers:
      - `Authorization`: `=Bearer {{ $item("0").$node["Create Session"].json["accessJwt"] }}`
      - `Atproto-Proxy`: `did:web:api.bsky.chat#bsky_chat`
    - Connect output of `Define welcome message` to input.

11. **Create an HTTP Request node to send the welcome message**
    - Name: `Send message`
    - Type: HTTP Request
    - Method: POST
    - URL: `={{ $item("0").$node["Create Session"].json.didDoc.service[0].serviceEndpoint }}/xrpc/chat.bsky.convo.sendMessage`
    - Headers:
      - `Authorization`: `=Bearer {{ $item("0").$node["Create Session"].json["accessJwt"] }}`
      - `Atproto-Proxy`: `did:web:api.bsky.chat#bsky_chat`
    - Body (JSON):
      ```json
      {
        "convoId": "{{ $json.convo.id }}",
        "message": {
          "text": "{{ $('Define welcome message').item.json.text }}\n\n{{ $('Define welcome message').item.json.link }}",
          "facets": [
            {
              "index": {
                "byteStart": {{ $('Define welcome message').item.json.text.length }},
                "byteEnd": {{ $('Define welcome message').item.json.text.length + 3 + $('Define welcome message').item.json.link.length }}
              },
              "features": [
                {
                  "$type": "app.bsky.richtext.facet#link",
                  "uri": "{{ $('Define welcome message').item.json.link }}"
                }
              ]
            }
          ]
        }
      }
      ```
    - Connect output of `Get conversation ID` to input.
    - Connect output back to `Loop Over Items` (to continue processing next follower).

12. **Create a Wait node**
    - Name: `Wait`
    - Type: Wait
    - Connect output of `List followers` to input.
    - This node allows for a delay to ensure all processing is complete before saving the new followers list.

13. **Create a Convert to File node**
    - Name: `Convert to File`
    - Type: Convert To File
    - Operation: toJson
    - File Name: `followers-basuracero.json` (or any temporary name)
    - Connect output of `Wait` to input.

14. **Create a Read/Write File node to save followers**
    - Name: `Save followers to file`
    - Type: Read/Write File
    - Operation: Write
    - File Name: `followers-{{ $('Create Session').item.json.handle }}.json`
    - Append: false (overwrite)
    - Connect output of `Convert to File` to input.

15. **Create a No Operation node**
    - Name: `No Operation, do nothing`
    - Type: NoOp
    - Connect output of `Save followers to file` to input.
    - Terminates the workflow cleanly.

16. **Add Sticky Notes for user instructions:**
    - Before `Create Session`: "### 1. Define your Bluesky user and app password first\n\nThe App password should have access to private messages"
    - Before `Define welcome message`: "### 2. Define your welcome message and link here"
    - Near `Save followers to file`: "### 3. **Important** \n\nYou need to manually run \"Save followers to file\" once before the first time so you populate your list of existing followers"
    - At workflow start: Overview note explaining the purpose and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| You need to create a [BlueSky app password](https://bsky.app/settings/app-passwords) with private messages access before running this workflow. | Official BlueSky App Password Creation page                                                                    |
| Adjust the check frequency carefully to avoid hitting the 100 createSession per day rate limit imposed by BlueSky API.                   | Rate limiting considerations                                                                                     |
| You must manually run the `Save followers to file` node once at setup to initialize the follower list file.                              | Workflow setup step                                                                                               |
| Feedback and improvements can be shared on the n8n community forums: [n8n BlueSky Workflow Discussion](https://community.n8n.io/t/bluesky-send-a-welcome-message-to-new-followers/62890) | Community forum for support and feedback                                                                         |
| Use the `Atproto-Proxy: did:web:api.bsky.chat#bsky_chat` header for chat-related API calls to ensure proper routing and compatibility.    | Required header for chat API endpoints                                                                           |

---

This comprehensive breakdown allows advanced users and AI agents to understand the workflow’s structure, reproduce it step-by-step, and anticipate potential errors or customization points.