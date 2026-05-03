Add, update, and fetch contacts from a Notion database by email

https://n8nworkflows.xyz/workflows/add--update--and-fetch-contacts-from-a-notion-database-by-email-14304


# Add, update, and fetch contacts from a Notion database by email

# 1. Workflow Overview

This workflow manages contacts stored in a Notion database and supports three operations driven by an input field named `action`:

- `create`: add a new contact if the email does not already exist
- `update`: find a contact by email and update its properties
- `get`: retrieve a contact by email

Its main use case is lightweight CRM-style contact management where email acts as the unique lookup key. The workflow is currently designed for manual execution with pinned JSON test data, but it is clearly intended to be adapted for API-style use with a Webhook trigger.

## 1.1 Input Reception and Routing

The workflow starts from a `Manual Trigger`, receives a JSON payload, and routes execution according to the `action` field via a `Switch` node.

## 1.2 Create Contact Branch

The create branch first queries Notion for an existing page with the same email. If a match exists, creation is blocked; otherwise, a new page is inserted into the Notion database.

## 1.3 Update Contact Branch

The update branch searches Notion by email, checks whether a matching contact exists, and if found updates the corresponding page using its Notion page ID.

## 1.4 Get Contact Branch

The get branch queries Notion by email and returns the first matching page’s properties/data.

## 1.5 Response Handling

Each branch ends by returning JSON through a `Respond to Webhook` node. Although these nodes are configured correctly for webhook-style behavior, the current workflow entry point is a `Manual Trigger`, so these response nodes are only fully meaningful once the trigger is replaced by a `Webhook`.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Routing

### Overview
This block accepts the incoming payload and decides which branch to execute based on `action`. It is the central dispatcher for the workflow.

### Nodes Involved
- Manual Trigger
- Switch1
- Sticky Note — Main
- Sticky Note — Switch

### Node Details

#### Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual workflow starter for testing and development.
- **Configuration choices:** No parameters are configured. The workflow includes pinned sample data on this node:
  - `name`: `Jane Doe`
  - `email`: `jane.doe@example.com`
  - `notes`: `Met at Berlin conference`
  - `phone`: `+49 123 456 789`
  - `action`: `create`
  - `status`: `Lead`
- **Key expressions or variables used:** Downstream nodes reference fields from this trigger payload such as `$json.action`, `$json.email`, and in one branch specifically `$('Manual Trigger').item.json.*`.
- **Input and output connections:** No input; outputs to `Switch1`.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - No payload validation occurs here.
  - If `action` is missing or misspelled, the downstream switch will not route to any branch.
  - Manual Trigger is not suitable for production/API use.
- **Sub-workflow reference:** None.

#### Switch1
- **Type and technical role:** `n8n-nodes-base.switch`; routes the item into one of three branches.
- **Configuration choices:** Three rules compare `{{ $json.action }}` to exact string values:
  - `create`
  - `update`
  - `get`
- **Key expressions or variables used:** `={{ $json.action }}`
- **Input and output connections:**
  - Input from `Manual Trigger`
  - Output 0 → `Search — Email Exists?`
  - Output 1 → `Search — Find by Email`
  - Output 2 → `Notion — Get Person1`
- **Version-specific requirements:** `typeVersion: 3`.
- **Edge cases or potential failure types:**
  - Exact string matching is case-sensitive due to strict comparison settings.
  - Unsupported actions produce no routed output and effectively stop execution silently unless additional fallback handling is added.
- **Sub-workflow reference:** None.

#### Sticky Note — Main
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation/annotation only.
- **Configuration choices:** Explains workflow behavior, setup steps, required Notion columns, and recommends replacing Manual Trigger with Webhook in production.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note — Switch
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation for the router.
- **Configuration choices:** States that `create`, `update`, and `get` are routed to separate branches.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Create Contact Branch

### Overview
This block prevents duplicate contacts by searching Notion for an existing email before attempting creation. If a record already exists, the request is rejected; otherwise, a new page is created in the configured Notion database.

### Nodes Involved
- Search — Email Exists?
- Already Exists?
- Block Duplicate
- Notion — Create Person1
- Sticky Note — Create

### Node Details

#### Search — Email Exists?
- **Type and technical role:** `n8n-nodes-base.notion`; searches the Notion database for an existing contact by email.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database: selected from credential-linked Notion databases
  - Filter type: manual
  - Condition: `Email|email` equals incoming `email`
  - Limit: `1`
  - `alwaysOutputData: true` is enabled
- **Key expressions or variables used:** `={{ $json.email }}`
- **Input and output connections:**
  - Input from `Switch1` create output
  - Output to `Already Exists?`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Notion credential/authentication failure
  - Integration not shared with the database
  - Database schema mismatch if column `Email` is missing or not of email type
  - If `email` is blank or undefined, the filter may return no match or behave unexpectedly
  - Because `alwaysOutputData` is enabled, the next node still runs even if no records are found
- **Sub-workflow reference:** None.

#### Already Exists?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the search returned a Notion page ID.
- **Configuration choices:** Condition tests whether `{{ $json.id }}` is not empty.
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:**
  - Input from `Search — Email Exists?`
  - True output → `Block Duplicate`
  - False output → `Notion — Create Person1`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Relies on search result shape containing `id`
  - If Notion response format changes or mapping is altered, duplicate detection may fail
- **Sub-workflow reference:** None.

#### Block Duplicate
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns an error JSON response when a duplicate exists.
- **Configuration choices:**
  - Response format: JSON
  - Response body:
    ```json
    { "error": "Contact with this email already exists. Use action: update instead." }
    ```
- **Key expressions or variables used:** Static JSON expression.
- **Input and output connections:**
  - Input from `Already Exists?` true branch
  - No downstream output
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - In a Manual Trigger context, this node is not useful for real HTTP responses
  - In webhook usage, it requires execution to have originated from a Webhook waiting for response
- **Sub-workflow reference:** None.

#### Notion — Create Person1
- **Type and technical role:** `n8n-nodes-base.notion`; creates a new contact page in Notion.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: create
  - Database: selected database
  - Title field set from the routed item’s `name`
  - Properties mapped:
    - `Email|email` ← email
    - `Phone|phone_number` ← phone
    - `Status|select` ← status
    - `Notes|rich_text` ← notes
- **Key expressions or variables used:**
  - `={{ $('Switch1').item.json.name }}`
  - `={{ $('Switch1').item.json.email }}`
  - `={{ $('Switch1').item.json.phone }}`
  - `={{ $('Switch1').item.json.status }}`
  - `={{ $('Switch1').item.json.notes }}`
- **Input and output connections:**
  - Input from `Already Exists?` false branch
  - Output to `Respond`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Credential failure or missing database access
  - Schema mismatch if required properties are absent or typed differently
  - `Status` select values must already exist in Notion unless Notion auto-creates them in your setup; otherwise update/create may fail
  - Use of `$('Switch1').item.json...` assumes consistent item linking; this is acceptable here but could become fragile if the branch is reworked with item transformations
- **Sub-workflow reference:** None.

#### Sticky Note — Create
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for create logic.
- **Configuration choices:** Explains duplicate prevention before creation.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.3 Update Contact Branch

### Overview
This block finds a contact by email, validates that a matching person exists, and updates the corresponding Notion page using the discovered page ID. It avoids requiring the caller to know the Notion page ID in advance.

### Nodes Involved
- Search — Find by Email
- Person Found?
- Not Found Error
- Notion — Update Person1
- Sticky Note — Update

### Node Details

#### Search — Find by Email
- **Type and technical role:** `n8n-nodes-base.notion`; searches for an existing contact by email for update purposes.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database: selected database
  - Manual filter: `Email|email` equals incoming `email`
  - Limit: `1`
- **Key expressions or variables used:** `={{ $json.email }}`
- **Input and output connections:**
  - Input from `Switch1` update output
  - Output to `Person Found?`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Same Notion auth/database-sharing issues as the create search node
  - If no match exists, the next IF node handles the absence
  - If multiple contacts share the same email, only the first match is updated because `limit` is 1
- **Sub-workflow reference:** None.

#### Person Found?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the search returned a page ID.
- **Configuration choices:** Condition tests if `{{ $json.id }}` is not empty.
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:**
  - Input from `Search — Find by Email`
  - True output → `Notion — Update Person1`
  - False output → `Not Found Error`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Depends on search response shape
  - If the search returns an unexpected data format, false negatives are possible
- **Sub-workflow reference:** None.

#### Not Found Error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns an error payload when no matching contact exists.
- **Configuration choices:**
  - Response format: JSON
  - Response body:
    ```json
    { "error": "No contact found with that email. Use action: create instead." }
    ```
- **Key expressions or variables used:** Static JSON expression.
- **Input and output connections:**
  - Input from `Person Found?` false branch
  - No downstream output
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** Same webhook-response caveat as other Respond nodes.
- **Sub-workflow reference:** None.

#### Notion — Update Person1
- **Type and technical role:** `n8n-nodes-base.notion`; updates the existing Notion page.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `update`
  - `pageId` uses the ID returned by the search node
  - Properties mapped:
    - `Name|title` ← manual trigger `name`
    - `Email|email` ← manual trigger `email`
    - `Phone|phone_number` ← manual trigger `phone`
    - `Status|select` ← manual trigger `status`
    - `Notes|rich_text` ← manual trigger `notes`
- **Key expressions or variables used:**
  - `={{ $json.id }}`
  - `={{ $('Manual Trigger').item.json.name }}`
  - `={{ $('Manual Trigger').item.json.email }}`
  - `={{ $('Manual Trigger').item.json.phone }}`
  - `={{ $('Manual Trigger').item.json.status }}`
  - `={{ $('Manual Trigger').item.json.notes }}`
- **Input and output connections:**
  - Input from `Person Found?` true branch
  - Output to `Respond`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - If the workflow is later migrated to Webhook without updating these expressions, they should still work if the trigger node name changes carefully; otherwise expressions will break
  - If some fields are omitted in the incoming payload, update behavior may overwrite properties with empty values depending on n8n/Notion handling
  - `Status` must be compatible with available select options
- **Sub-workflow reference:** None.

#### Sticky Note — Update
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for update logic.
- **Configuration choices:** Notes that lookup is by email and no manual page ID is required.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.4 Get Contact Branch

### Overview
This block retrieves contact data by email from Notion and forwards the resulting page data to the final response node.

### Nodes Involved
- Notion — Get Person1
- Sticky Note — Get

### Node Details

#### Notion — Get Person1
- **Type and technical role:** `n8n-nodes-base.notion`; fetches a matching contact from the Notion database.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database: selected database
  - Filter: `Email|email` equals incoming `email`
  - Limit: `1`
- **Key expressions or variables used:** `={{ $json.email }}`
- **Input and output connections:**
  - Input from `Switch1` get output
  - Output to `Respond`
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - If no record exists, the node may output no items; in that case the downstream response may not run depending on execution behavior
  - If duplicate emails exist, only the first result is returned
  - Same auth/schema/database-sharing issues as other Notion nodes
- **Sub-workflow reference:** None.

#### Sticky Note — Get
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for retrieval logic.
- **Configuration choices:** States that the branch returns all Notion page properties.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.5 Response Handling

### Overview
This block sends the resulting payload back as JSON. It is shared by create, update, and get success paths.

### Nodes Involved
- Respond

### Node Details

#### Respond
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns the current item JSON as the HTTP response body.
- **Configuration choices:**
  - Response format: JSON
  - Body: `={{ $json }}`
- **Key expressions or variables used:** `={{ $json }}`
- **Input and output connections:**
  - Inputs from:
    - `Notion — Create Person1`
    - `Notion — Update Person1`
    - `Notion — Get Person1`
  - No downstream output
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - Requires a Webhook-originated execution to behave as intended
  - In current manual mode, this is effectively a placeholder for future API use
  - The output is raw Notion item JSON, which may contain more metadata than desired for external consumers
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Starts the workflow manually with test JSON data |  | Switch1 | ## How it works<br>1. The Manual Trigger fires with a JSON payload containing an `action` field (`create`, `update`, or `get`).<br>2. The Switch node routes the request to the correct branch.<br>3. **Create** — searches by email first. If the contact exists, the request is blocked. If not, a new Notion page is created.<br>4. **Update** — searches by email to find the page ID automatically, then updates the matching row.<br>5. **Get** — searches by email and returns the contact's full data.<br><br>## Setup steps<br>1. Connect your Notion account under Credentials.<br>2. Open each Notion node and select your database from the list.<br>3. In Notion, go to your database → `...` → Connections → add your integration.<br>4. Make sure your database has these columns: `Name` (title), `Email` (email), `Phone` (phone), `Status` (select), `Notes` (text).<br>5. For production use, replace the Manual Trigger with a Webhook node. |
| Switch1 | Switch | Routes execution based on the `action` field | Manual Trigger | Search — Email Exists?; Search — Find by Email; Notion — Get Person1 | ## Router<br>Routes `create`, `update`, and `get`<br>to their respective branches. |
| Search — Email Exists? | Notion | Looks up an existing contact by email before creation | Switch1 | Already Exists? | ## Create branch<br>Checks if email already exists before creating.<br>Blocks duplicates automatically. |
| Already Exists? | If | Detects whether the email search returned a match | Search — Email Exists? | Block Duplicate; Notion — Create Person1 | ## Create branch<br>Checks if email already exists before creating.<br>Blocks duplicates automatically. |
| Block Duplicate | Respond to Webhook | Returns duplicate-contact error | Already Exists? |  | ## Create branch<br>Checks if email already exists before creating.<br>Blocks duplicates automatically. |
| Notion — Create Person1 | Notion | Creates a new Notion contact page | Already Exists? | Respond | ## Create branch<br>Checks if email already exists before creating.<br>Blocks duplicates automatically. |
| Search — Find by Email | Notion | Finds an existing contact by email for update | Switch1 | Person Found? | ## Update branch<br>Finds contact by email automatically.<br>No manual page ID needed. |
| Person Found? | If | Checks whether the contact exists before update | Search — Find by Email | Notion — Update Person1; Not Found Error | ## Update branch<br>Finds contact by email automatically.<br>No manual page ID needed. |
| Not Found Error | Respond to Webhook | Returns not-found error for update requests | Person Found? |  | ## Update branch<br>Finds contact by email automatically.<br>No manual page ID needed. |
| Notion — Update Person1 | Notion | Updates the matched Notion page | Person Found? | Respond | ## Update branch<br>Finds contact by email automatically.<br>No manual page ID needed. |
| Notion — Get Person1 | Notion | Retrieves a contact by email | Switch1 | Respond | ## Get branch<br>Retrieves a contact by email.<br>Returns all Notion page properties. |
| Respond | Respond to Webhook | Returns successful branch output as JSON | Notion — Create Person1; Notion — Update Person1; Notion — Get Person1 |  |  |
| Sticky Note — Main | Sticky Note | Documents workflow purpose and setup |  |  |  |
| Sticky Note — Create | Sticky Note | Documents create branch behavior |  |  |  |
| Sticky Note — Update | Sticky Note | Documents update branch behavior |  |  |  |
| Sticky Note — Get | Sticky Note | Documents get branch behavior |  |  |  |
| Sticky Note — Switch | Sticky Note | Documents routing behavior |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: **Add, update, and fetch contacts from a Notion database using n8n**.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Leave default settings.
   - Optionally pin sample JSON for testing:
     - `name`: `Jane Doe`
     - `email`: `jane.doe@example.com`
     - `notes`: `Met at Berlin conference`
     - `phone`: `+49 123 456 789`
     - `action`: `create`
     - `status`: `Lead`

3. **Add the main router**
   - Node type: **Switch**
   - Name it `Switch1`.
   - Configure three outputs using rules on `{{$json.action}}`:
     1. Equals `create`
     2. Equals `update`
     3. Equals `get`
   - Connect:
     - `Manual Trigger` → `Switch1`

4. **Create the create-branch search node**
   - Node type: **Notion**
   - Name: `Search — Email Exists?`
   - Credentials: connect your **Notion** credential.
   - Resource: **Database Page**
   - Operation: **Get Many / Get All**
   - Select your database.
   - Filter type: **Manual**
   - Add condition:
     - Property: `Email`
     - Type: email
     - Condition: equals
     - Value: `{{$json.email}}`
   - Set **Limit = 1**
   - Enable **Always Output Data**
   - Connect:
     - `Switch1` create output → `Search — Email Exists?`

5. **Add duplicate detection**
   - Node type: **If**
   - Name: `Already Exists?`
   - Condition:
     - Left value: `{{$json.id}}`
     - Operator: **is not empty**
   - Connect:
     - `Search — Email Exists?` → `Already Exists?`

6. **Add duplicate error response**
   - Node type: **Respond to Webhook**
   - Name: `Block Duplicate`
   - Respond with: **JSON**
   - Response body:
     ```json
     { "error": "Contact with this email already exists. Use action: update instead." }
     ```
   - Connect:
     - `Already Exists?` true output → `Block Duplicate`

7. **Add create-contact node**
   - Node type: **Notion**
   - Name: `Notion — Create Person1`
   - Credentials: same Notion credential
   - Resource: **Database Page**
   - Operation: **Create**
   - Select the same database.
   - Set the title/name field to:
     - `{{$('Switch1').item.json.name}}`
   - Add properties:
     - `Email` (email) → `{{$('Switch1').item.json.email}}`
     - `Phone` (phone number) → `{{$('Switch1').item.json.phone}}`
     - `Status` (select) → `{{$('Switch1').item.json.status}}`
     - `Notes` (rich text/text) → `{{$('Switch1').item.json.notes}}`
   - Connect:
     - `Already Exists?` false output → `Notion — Create Person1`

8. **Create the update-branch search node**
   - Node type: **Notion**
   - Name: `Search — Find by Email`
   - Credentials: same Notion credential
   - Resource: **Database Page**
   - Operation: **Get Many / Get All**
   - Select the same database.
   - Filter type: **Manual**
   - Add condition:
     - Property: `Email`
     - Type: email
     - Condition: equals
     - Value: `{{$json.email}}`
   - Set **Limit = 1**
   - Connect:
     - `Switch1` update output → `Search — Find by Email`

9. **Add update existence check**
   - Node type: **If**
   - Name: `Person Found?`
   - Condition:
     - Left value: `{{$json.id}}`
     - Operator: **is not empty**
   - Connect:
     - `Search — Find by Email` → `Person Found?`

10. **Add update not-found response**
    - Node type: **Respond to Webhook**
    - Name: `Not Found Error`
    - Respond with: **JSON**
    - Response body:
      ```json
      { "error": "No contact found with that email. Use action: create instead." }
      ```
    - Connect:
      - `Person Found?` false output → `Not Found Error`

11. **Add update-contact node**
    - Node type: **Notion**
    - Name: `Notion — Update Person1`
    - Credentials: same Notion credential
    - Resource: **Database Page**
    - Operation: **Update**
    - Page ID:
      - `{{$json.id}}`
    - Map properties:
      - `Name` (title) → `{{$('Manual Trigger').item.json.name}}`
      - `Email` (email) → `{{$('Manual Trigger').item.json.email}}`
      - `Phone` (phone number) → `{{$('Manual Trigger').item.json.phone}}`
      - `Status` (select) → `{{$('Manual Trigger').item.json.status}}`
      - `Notes` (rich text/text) → `{{$('Manual Trigger').item.json.notes}}`
    - Connect:
      - `Person Found?` true output → `Notion — Update Person1`

12. **Create the get branch**
    - Node type: **Notion**
    - Name: `Notion — Get Person1`
    - Credentials: same Notion credential
    - Resource: **Database Page**
    - Operation: **Get Many / Get All**
    - Select the same database.
    - Filter type: **Manual**
    - Add condition:
      - Property: `Email`
      - Type: email
      - Condition: equals
      - Value: `{{$json.email}}`
    - Set **Limit = 1**
    - Connect:
      - `Switch1` get output → `Notion — Get Person1`

13. **Add the shared success response node**
    - Node type: **Respond to Webhook**
    - Name: `Respond`
    - Respond with: **JSON**
    - Response body:
      - `{{$json}}`
    - Connect:
      - `Notion — Create Person1` → `Respond`
      - `Notion — Update Person1` → `Respond`
      - `Notion — Get Person1` → `Respond`

14. **Add documentation sticky notes**
    - Add five **Sticky Note** nodes and place them near the related areas.
    - Suggested contents:

   **Main**
   - Explain overall flow
   - Mention required Notion database columns:
     - `Name` (title)
     - `Email` (email)
     - `Phone` (phone)
     - `Status` (select)
     - `Notes` (text/rich text)
   - Mention production recommendation: replace Manual Trigger with Webhook

   **Switch**
   - Explain router purpose

   **Create**
   - Explain duplicate email prevention

   **Update**
   - Explain lookup by email before update

   **Get**
   - Explain retrieval by email

15. **Configure Notion credentials**
   - Create or select a **Notion API credential** in n8n.
   - In Notion, open the target database.
   - Use **Connections** to add your integration to the database.
   - Confirm the database schema includes:
     - `Name` as the title property
     - `Email` as email
     - `Phone` as phone number
     - `Status` as select
     - `Notes` as rich text or text-compatible field

16. **Test the create path**
   - Pin or send JSON with:
     - `action: create`
     - plus `name`, `email`, `phone`, `status`, `notes`
   - Execute manually.
   - Confirm:
     - duplicate emails return the duplicate error branch
     - new emails create a page

17. **Test the update path**
   - Change payload to:
     - `action: update`
     - same email as an existing contact
     - updated values for other fields
   - Execute manually.
   - Confirm the matching Notion page is updated.

18. **Test the get path**
   - Change payload to:
     - `action: get`
     - `email` for an existing contact
   - Execute manually.
   - Confirm the first matching Notion page is returned.

19. **Adapt for production**
   - Replace `Manual Trigger` with a **Webhook** node.
   - Update expressions that explicitly reference `Manual Trigger` if you rename or remove it:
     - especially in `Notion — Update Person1`
   - Keep `Respond to Webhook` nodes as the branch outputs.
   - Consider adding:
     - validation for missing fields
     - default/fallback branch for unknown actions
     - normalized output instead of raw Notion payloads

### Sub-workflow setup
This workflow does **not** use any sub-workflow nodes and does not invoke other workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is currently built for manual testing but is clearly intended for webhook/API use in production. | Replace `Manual Trigger` with a `Webhook` node when exposing it externally. |
| The Notion integration must be explicitly added to the target database, otherwise search/create/update operations will fail even if credentials are valid. | In Notion database menu: `...` → `Connections` |
| Email is treated as the unique lookup key, but Notion does not enforce uniqueness automatically. Duplicate rows could still exist if created outside this workflow or through race conditions. | Design consideration |
| The get and update searches use `limit: 1`, so if duplicates exist, only the first matching record will be returned or updated. | Design limitation |
| The workflow returns raw Notion node output on success. You may want to add a Set/Edit Fields node to return a cleaner API schema. | Recommended enhancement |