Convert Slack Messages to Notion Todos Using 📝 Emoji Reactions

https://n8nworkflows.xyz/workflows/convert-slack-messages-to-notion-todos-using----emoji-reactions-9009


# Convert Slack Messages to Notion Todos Using 📝 Emoji Reactions

### 1. Workflow Overview

This workflow automates the process of converting Slack messages into actionable todos in Notion by using a specific emoji reaction (📝, represented as "memo" in Slack). It also aggregates and sends daily reminders of unchecked todos back to Slack.

The workflow is logically divided into two main blocks:

- **1.1 Reaction-based ToDo Creation:** Listens for "reaction_added" events in Slack, filters for the 📝 emoji, retrieves the associated message, and creates a corresponding ToDo item in Notion.

- **1.2 Daily ToDo Aggregation and Slack Notification:** Runs every day at 8 AM to fetch all todos from Notion, filters unchecked todos, aggregates them into a summary, and sends this summary as a Slack message.

---

### 2. Block-by-Block Analysis

#### 2.1 Reaction-based ToDo Creation

**Overview:**  
This block triggers on Slack emoji reactions and, if the emoji is 📝, retrieves the reacted message and saves it as a ToDo item in Notion.

**Nodes Involved:**  
- Slack Trigger  
- Filter by emoji  
- Get reacted message  
- Added message to Notion  
- Sticky Note (documentation)

**Node Details:**

- **Slack Trigger**  
  - *Type:* Slack Trigger  
  - *Role:* Listens for Slack events, specifically "reaction_added" in the entire workspace.  
  - *Configuration:* Watches all workspace reactions, resolves IDs to objects, triggers on `reaction_added`.  
  - *Input:* None (event-based trigger)  
  - *Output:* Event data including reaction details, message timestamp, user info.  
  - *Edge cases:* Missing permissions for Slack events, webhook misconfiguration, Slack API rate limits.

- **Filter by emoji**  
  - *Type:* Filter  
  - *Role:* Passes only events where the reaction emoji equals "memo" (the 📝 emoji).  
  - *Configuration:* Condition is `$json.reaction === "memo"` (case-sensitive, strict).  
  - *Input:* Slack Trigger output  
  - *Output:* Passes filtered events to next node; blocks others.  
  - *Edge cases:* Reaction emoji not present or different emoji used.

- **Get reacted message**  
  - *Type:* Slack  
  - *Role:* Retrieves the full message data from the channel and timestamp associated with the reaction.  
  - *Configuration:* Gets reaction resource with operation "get", uses channelId `"general"` (cached), timestamp from `$json.item.ts`, OAuth2 authentication.  
  - *Input:* Filter by emoji output  
  - *Output:* Full message JSON including text and permalink.  
  - *Edge cases:* Message deleted or inaccessible, permission errors, invalid timestamp, channel mismatch.

- **Added message to Notion**  
  - *Type:* Notion  
  - *Role:* Creates a new ToDo block in Notion with the reacted Slack message text and permalink.  
  - *Configuration:*  
    - Resource: block  
    - Block type: to_do  
    - Text content: Slack message text (`$json.message.text`) plus message permalink as a clickable link.  
  - *Input:* Get reacted message output  
  - *Output:* Confirmation of block creation.  
  - *Edge cases:* Notion API authentication failures, quota limits, invalid block structure.

- **Sticky Note**  
  - *Type:* Sticky Note (documentation)  
  - *Role:* Describes the reaction-to-Notion message flow.  
  - *Content:* "Get reacted message — This is triggered by a reaction to a message, which then filters by the emoji selected and gets the final message."

---

#### 2.2 Daily ToDo Aggregation and Slack Notification

**Overview:**  
This block runs daily at 8 AM, retrieves all ToDo blocks from Notion, filters unchecked todos, aggregates their text content, and sends the aggregated list as a message to Slack users.

**Nodes Involved:**  
- Schedule Trigger  
- Get all todos from Notion  
- Filter todo  
- Filter not checked todos  
- Aggregate  
- Send a message  
- Sticky Notes (documentation)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the flow daily at 8 AM.  
  - *Configuration:* Interval trigger set for 8:00 AM every day.  
  - *Input:* None  
  - *Output:* Trigger event once daily.  
  - *Edge cases:* Workflow not activated, time zone mismatches.

- **Get all todos from Notion**  
  - *Type:* Notion  
  - *Role:* Fetches all blocks from the configured Notion document, including nested blocks.  
  - *Configuration:*  
    - Resource: block  
    - Operation: getAll  
    - Return all results, no pagination  
    - Fetch nested blocks enabled  
  - *Input:* Schedule Trigger output  
  - *Output:* Array of all blocks from Notion workspace.  
  - *Edge cases:* Notion API rate limits, invalid credentials, large data sets causing timeouts.

- **Filter todo**  
  - *Type:* Filter  
  - *Role:* Filters blocks to only those of type `to_do`.  
  - *Configuration:* `$json.type === "to_do"` (strict and case-sensitive).  
  - *Input:* Get all todos from Notion output  
  - *Output:* Passes only to_do blocks forward.  
  - *Edge cases:* Blocks missing type property, different block types.

- **Filter not checked todos**  
  - *Type:* Filter  
  - *Role:* Keeps only todos where `to_do.checked` is false (unchecked items).  
  - *Configuration:* `$json.to_do.checked === false` (boolean false).  
  - *Input:* Filter todo output  
  - *Output:* Passes unchecked todos forward.  
  - *Edge cases:* Missing checked property, inconsistent boolean representation.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Collects the text content from all unchecked todos into a single aggregated list.  
  - *Configuration:* Aggregates the field `to_do.text[0].text.content` from each item.  
  - *Input:* Filter not checked todos output  
  - *Output:* Aggregated list of todo texts.  
  - *Edge cases:* Empty todo lists, nested text arrays missing expected structure.

- **Send a message**  
  - *Type:* Slack  
  - *Role:* Sends the aggregated ToDo text as a message to the Slack user who triggered the workflow or a preset user.  
  - *Configuration:*  
    - Text: Uses aggregated content `$json.content`  
    - Select: user (direct message)  
    - Authentication: OAuth2  
  - *Input:* Aggregate output  
  - *Output:* Confirmation of Slack message sent.  
  - *Edge cases:* User ID missing or invalid, Slack API failures, authentication errors.

- **Sticky Notes**  
  - *Sticky Note2:* "Get todos daily — Every day, this flow runs to pull the todos from the Notion document and send them as a Slack message."  
  - *Sticky Note3:* "Message filter — Since we don't want to return all the messages, we filter them accordingly and send them to Slack."

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                              | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                          |
|------------------------|---------------------|----------------------------------------------|---------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| Slack Trigger          | Slack Trigger       | Triggers on Slack "reaction_added" events    | None                      | Filter by emoji             | ## Get reacted message This is triggered by a reaction to a message, which then filters by emoji.  |
| Filter by emoji         | Filter              | Filters reactions by emoji "memo" (📝)        | Slack Trigger             | Get reacted message         | Same as above                                                                                       |
| Get reacted message     | Slack               | Retrieves the Slack message for reaction      | Filter by emoji           | Added message to Notion     | Same as above                                                                                       |
| Added message to Notion | Notion              | Adds Slack message as a ToDo block in Notion | Get reacted message       |                            | ## Save to Notion The message is then saved to a Notion document                                   |
| Schedule Trigger        | Schedule Trigger    | Triggers flow daily at 8 AM                    | None                      | Get all todos from Notion   | ## Get todos daily Every day, this flow runs to pull the todos from Notion and send them to Slack  |
| Get all todos from Notion| Notion             | Retrieves all blocks (todos) from Notion      | Schedule Trigger          | Filter todo                 | Same as above                                                                                       |
| Filter todo             | Filter              | Filters blocks to type "to_do"                 | Get all todos from Notion | Filter not checked todos    | ## Message filter Since we don't want to return all the messages, we filter them accordingly       |
| Filter not checked todos| Filter              | Filters todos where checked == false           | Filter todo               | Aggregate                  | Same as above                                                                                       |
| Aggregate               | Aggregate           | Aggregates text content of unchecked todos    | Filter not checked todos  | Send a message             | Same as above                                                                                       |
| Send a message          | Slack               | Sends aggregated todo list message to Slack   | Aggregate                 |                            | Same as above                                                                                       |
| Sticky Note             | Sticky Note         | Documentation node                             | None                      | None                       | ## Get reacted message This is triggered by a reaction to a message, which then filters by emoji.  |
| Sticky Note1            | Sticky Note         | Documentation node                             | None                      | None                       | ## Save to Notion The message is then saved to a Notion document                                   |
| Sticky Note2            | Sticky Note         | Documentation node                             | None                      | None                       | ## Get todos daily Every day, this flow runs to pull the todos from the Notion document and sends. |
| Sticky Note3            | Sticky Note         | Documentation node                             | None                      | None                       | ## Message filter Since we don't want to return all the messages, we filter them accordingly       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger node**  
   - Type: Slack Trigger  
   - Parameters:  
     - Trigger: reaction_added  
     - Watch Workspace: true  
     - Options: resolve IDs enabled  
   - Credentials: Slack OAuth2 with appropriate permissions to read reactions and messages.

2. **Add Filter node: Filter by emoji**  
   - Type: Filter  
   - Condition: `$json.reaction === "memo"` (case-sensitive, strict equality)  
   - Connect input from Slack Trigger node.

3. **Add Slack node: Get reacted message**  
   - Type: Slack  
   - Resource: reaction  
   - Operation: get  
   - Channel ID: use channel ID from event data (`$json.item.channel` or cached "general")  
   - Timestamp: reaction item timestamp (`$json.item.ts`)  
   - Authentication: OAuth2 (same as Slack Trigger)  
   - Connect input from Filter by emoji node.

4. **Add Notion node: Added message to Notion**  
   - Type: Notion  
   - Resource: block  
   - Operation: create (implied)  
   - Block UI:  
     - Type: to_do  
     - Text:  
       - First text: Slack message text (`$json.message.text`)  
       - Second text: Slack message permalink as clickable link (`$json.message.permalink`)  
   - Credentials: Notion OAuth with write access to target page/database  
   - Connect input from Get reacted message node.

5. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters:  
     - Rule: interval trigger at 8:00 AM every day  
   - No credentials needed.  

6. **Add Notion node: Get all todos from Notion**  
   - Type: Notion  
   - Resource: block  
   - Operation: getAll  
   - Return all: true  
   - Fetch nested blocks: true  
   - Credentials: Notion OAuth with read access  
   - Connect input from Schedule Trigger node.

7. **Add Filter node: Filter todo**  
   - Type: Filter  
   - Condition: `$json.type === "to_do"` (strict and case-sensitive)  
   - Connect input from Get all todos from Notion node.

8. **Add Filter node: Filter not checked todos**  
   - Type: Filter  
   - Condition: `$json.to_do.checked === false` (boolean false)  
   - Connect input from Filter todo node.

9. **Add Aggregate node**  
   - Type: Aggregate  
   - Field to aggregate: `to_do.text[0].text.content`  
   - Connect input from Filter not checked todos node.

10. **Add Slack node: Send a message**  
    - Type: Slack  
    - Text: aggregated content (`$json.content`)  
    - Select: user (direct message)  
    - Authentication: OAuth2 (same Slack credentials)  
    - Connect input from Aggregate node.

11. **Add Sticky Notes for documentation** (optional)  
    - Add sticky notes near each block describing their purpose as per the content above.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow converts Slack emoji reactions (📝) into actionable Notion ToDos and sends daily reminders.     | Core functionality overview.                                                                     |
| Slack OAuth2 credentials need permissions for reading reactions and messages, and sending messages.      | Slack API permissions required.                                                                  |
| Notion OAuth credentials need read/write access to the target page or database blocks used for todos.     | Notion API authentication and permissions.                                                      |
| The emoji filter uses the Slack reaction name "memo" for the 📝 emoji.                                    | Slack reaction naming conventions.                                                              |
| Aggregation node assumes the first rich text object in to_do.text contains relevant content.              | Potential edge case if Notion blocks structure changes.                                          |
| Time zone considerations for Schedule Trigger should be checked to match user expectations.              | Important for daily 8 AM trigger accuracy.                                                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.