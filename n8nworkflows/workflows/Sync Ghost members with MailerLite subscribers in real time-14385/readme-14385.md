Sync Ghost members with MailerLite subscribers in real time

https://n8nworkflows.xyz/workflows/sync-ghost-members-with-mailerlite-subscribers-in-real-time-14385


# Sync Ghost members with MailerLite subscribers in real time

# 1. Workflow Overview

This workflow synchronizes **Ghost members** with **MailerLite subscribers** in near real time using **Ghost webhooks** and an **n8n webhook listener**.

It has two main purposes:

1. Provide a **configuration form** where an operator can define:
   - the Ghost Admin API base URL,
   - the callback webhook URL,
   - whether member-added sync is enabled,
   - whether member-deleted sync is enabled.

2. Automatically:
   - create or remove the corresponding Ghost webhooks,
   - persist the webhook IDs in an n8n Data Table,
   - update the workflow’s own form defaults so the current sync state is remembered,
   - react to incoming Ghost webhook events and update MailerLite accordingly.

## 1.1 Configuration Intake and Initialization

The workflow starts from a **Form Trigger**. It receives sync settings from the user, checks whether a dedicated Data Table exists, and creates it if necessary.

## 1.2 Sync Intent Derivation

The submitted form values are transformed into an internal list of desired webhook states:
- `member.added` → enabled/disabled
- `member.deleted` → enabled/disabled

This list is split so each event can be processed independently.

## 1.3 Ghost Webhook Management

For each event type, the workflow either:
- creates a webhook in Ghost and stores its ID, or
- retrieves the existing stored webhook ID, deletes the webhook in Ghost, and removes the stored row.

## 1.4 State Persistence in the Workflow Definition

After webhook changes, the workflow fetches its own definition through the n8n node, modifies the default values of the form fields, and updates itself. This allows the form to display the latest known settings on future loads.

## 1.5 Real-Time Sync Listener

A separate **Webhook** node receives Ghost event calls. It determines whether the payload represents a member deletion or not:
- if deleted, it unsubscribes the subscriber in MailerLite,
- otherwise, it creates a subscriber in MailerLite.

---

# 2. Block-by-Block Analysis

## 2.1 Introductory Documentation Block

**Overview:**  
This block contains visual notes describing the workflow’s purpose, setup instructions, and operational areas. These notes do not affect execution but are important for users rebuilding or maintaining the workflow.

**Nodes Involved:**
- `Sticky Note`
- `Sticky Note - Configuration`
- `Sticky Note - Webhook Management`
- `Sticky Note - Sync Listener`
- `Sticky Note - State Persistence`

### Node Details

#### Sticky Note
- **Type and role:** Sticky Note; overview banner for the whole workflow.
- **Configuration choices:** Describes the workflow as a real-time Ghost-to-MailerLite sync with independent add/remove controls.
- **Key expressions or variables used:** None.
- **Input/output connections:** None.
- **Version-specific requirements:** Standard sticky note support.
- **Edge cases or failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note - Configuration
- **Type and role:** Sticky Note; documents required credentials and setup.
- **Configuration choices:** Explains that credentials must be configured for MailerLite, Ghost, and n8n; also explains form fields.
- **Key expressions or variables used:** None.
- **Input/output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note - Webhook Management
- **Type and role:** Sticky Note; documents create/delete logic for Ghost webhooks.
- **Configuration choices:** Explains that enabling creates a Ghost webhook and stores its ID, while disabling deletes it and removes the stored ID.
- **Key expressions or variables used:** None.
- **Input/output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note - Sync Listener
- **Type and role:** Sticky Note; documents incoming webhook behavior.
- **Configuration choices:** Explains add → create subscriber and delete → unsubscribe subscriber.
- **Key expressions or variables used:** None.
- **Input/output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note - State Persistence
- **Type and role:** Sticky Note; documents self-update behavior.
- **Configuration choices:** Explains fetch-current-workflow, update-form-defaults, and save-back pattern.
- **Key expressions or variables used:** None.
- **Input/output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Configuration Intake and Data Table Initialization

**Overview:**  
This block accepts user-supplied configuration through a form, checks whether the `ghost_webhooks` Data Table exists, and creates it when absent. The block ensures later webhook management can persist Ghost webhook IDs safely.

**Nodes Involved:**
- `Sync Settings Form`
- `List data tables`
- `Check if ghost_webhooks table exists`
- `Create ghost_webhooks table`
- `Merge`

### Node Details

#### Sync Settings Form
- **Type and role:** `formTrigger`; entry point for manual configuration.
- **Configuration choices:**
  - Form title: `Sync Settings Form`
  - Submit button label: `Update Sync`
  - Fields:
    - `Ghost API URL`
    - `Webhook URL`
    - `Ghost Member Added > Added to Mailerlite` (radio, default `Disabled`)
    - `Ghost Member Removed > Removed from Mailerlite` (radio, default `Disabled`)
- **Key expressions or variables used:** Downstream nodes reference submitted values by exact field labels, for example:
  - `$('Sync Settings Form').first().json['Ghost API URL']`
  - `$('Sync Settings Form').first().json['Webhook URL']`
- **Input/output connections:** Starts the configuration branch; outputs to `List data tables`.
- **Version-specific requirements:** `typeVersion 2.5`; available in newer n8n versions with Forms.
- **Edge cases or failure types:**
  - Field labels are used as keys downstream; renaming them will break expressions.
  - Invalid Ghost URL formatting may break Ghost API requests later.
  - Wrong Webhook URL causes Ghost to send events to the wrong endpoint.
- **Sub-workflow reference:** None.

#### List data tables
- **Type and role:** `dataTable`; lists available Data Tables.
- **Configuration choices:** Resource set to `table`, operation lists tables; `alwaysOutputData` enabled.
- **Key expressions or variables used:** None directly.
- **Input/output connections:** Input from `Sync Settings Form`; output to `Check if ghost_webhooks table exists`.
- **Version-specific requirements:** `typeVersion 1.1`; requires Data Tables support in n8n.
- **Edge cases or failure types:**
  - Fails if Data Tables feature is unavailable in the environment.
  - Permission issues may block table listing.
- **Sub-workflow reference:** None.

#### Check if ghost_webhooks table exists
- **Type and role:** `if`; decides whether table creation should happen.
- **Configuration choices:** Tests whether `$json` is not empty.
- **Key expressions or variables used:**
  - `={{ $json }}`
- **Input/output connections:**
  - Input from `List data tables`
  - True output to `Merge`
  - False output to `Create ghost_webhooks table`
- **Version-specific requirements:** `typeVersion 2.3`.
- **Edge cases or failure types:**
  - The condition only checks non-empty output, not specifically whether a table named `ghost_webhooks` exists.
  - If any table exists, this may route to `Merge` even if `ghost_webhooks` itself does not exist.
  - In environments with existing unrelated tables, later operations on `ghost_webhooks` may fail if it was never created.
- **Sub-workflow reference:** None.

#### Create ghost_webhooks table
- **Type and role:** `dataTable`; creates the persistence table for Ghost webhook IDs.
- **Configuration choices:**
  - Operation: `create`
  - Table name: `ghost_webhooks`
  - Columns:
    - `webhook_id`
    - `event`
- **Key expressions or variables used:** None.
- **Input/output connections:** Input from false branch of `Check if ghost_webhooks table exists`; output to `Merge`.
- **Version-specific requirements:** `typeVersion 1.1`.
- **Edge cases or failure types:**
  - Table creation fails if a table with the same name already exists.
  - Requires table-creation permissions.
- **Sub-workflow reference:** None.

#### Merge
- **Type and role:** `merge`; rejoins the branch where the table already existed with the branch where it was newly created.
- **Configuration choices:** Default merge behavior; used here as a convergence point.
- **Key expressions or variables used:** None.
- **Input/output connections:**
  - Input 0 from `Check if ghost_webhooks table exists`
  - Input 1 from `Create ghost_webhooks table`
  - Output to `Identity syncs to be enabled/disabled`
- **Version-specific requirements:** `typeVersion 3.2`.
- **Edge cases or failure types:**
  - Merge behavior depends on n8n defaults; if multiple items appear unexpectedly, downstream code may process differently than intended.
- **Sub-workflow reference:** None.

---

## 2.3 Sync Intent Derivation

**Overview:**  
This block converts the form selections into a normalized internal array of webhook instructions, then splits them into one item per Ghost event type. It acts as the control plane for enabling or disabling each sync independently.

**Nodes Involved:**
- `Identity syncs to be enabled/disabled`
- `Split Out`
- `If sync enabled`

### Node Details

#### Identity syncs to be enabled/disabled
- **Type and role:** `code`; builds the desired state list for Ghost webhook synchronization.
- **Configuration choices:** For each incoming item, creates a `webhooks` array with two objects:
  - `{ type: 'member.added', action: 'enabled'|'disabled' }`
  - `{ type: 'member.deleted', action: 'enabled'|'disabled' }`
- **Key expressions or variables used:**
  - `$('Sync Settings Form').first().json['Ghost Member Added > Added to Mailerlite']`
  - `$('Sync Settings Form').first().json['Ghost Member Removed > Removed from Mailerlite']`
- **Input/output connections:** Input from `Merge`; output to `Split Out`.
- **Version-specific requirements:** `code` node `typeVersion 2`.
- **Edge cases or failure types:**
  - Depends on exact form field labels.
  - If the form payload is absent or malformed, the code throws errors.
  - Typo in node name or field label breaks expressions.
- **Sub-workflow reference:** None.

#### Split Out
- **Type and role:** `splitOut`; emits one item per desired webhook instruction.
- **Configuration choices:** Splits the field `webhooks`.
- **Key expressions or variables used:**
  - `fieldToSplitOut: webhooks`
- **Input/output connections:** Input from `Identity syncs to be enabled/disabled`; output to `If sync enabled`.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or failure types:**
  - If `webhooks` is not an array, the node fails or outputs unexpectedly.
- **Sub-workflow reference:** None.

#### If sync enabled
- **Type and role:** `if`; routes each event item based on whether it should be enabled or disabled.
- **Configuration choices:** Checks whether `$json.action` equals `enabled`.
- **Key expressions or variables used:**
  - `={{ $json.action }}`
- **Input/output connections:**
  - Input from `Split Out`
  - True branch to `Add webhook in Ghost`
  - False branch to `Get webhook ID to delete`
- **Version-specific requirements:** `typeVersion 2.3`.
- **Edge cases or failure types:**
  - If `action` is missing, items will likely fall to the false branch.
- **Sub-workflow reference:** None.

---

## 2.4 Ghost Webhook Creation Path

**Overview:**  
When a sync direction is enabled, this block creates the corresponding Ghost webhook pointing to the n8n webhook listener and stores the returned Ghost webhook ID in a Data Table.

**Nodes Involved:**
- `Add webhook in Ghost`
- `Store Ghost webhook ID`

### Node Details

#### Add webhook in Ghost
- **Type and role:** `httpRequest`; calls the Ghost Admin API to create a webhook.
- **Configuration choices:**
  - Method: `POST`
  - URL: `{{ Ghost API URL }}/ghost/api/v2/admin/webhooks`
  - Authentication: predefined credential type `ghostAdminApi`
  - JSON body contains:
    - `event`: current split item’s type
    - `target_url`: submitted webhook URL
- **Key expressions or variables used:**
  - `={{ $('Sync Settings Form').item.json['Ghost API URL'] }}/ghost/api/v2/admin/webhooks`
  - In JSON body:
    - `{{ $('Split Out').item.json.type }}`
    - `{{ $('Sync Settings Form').item.json['Webhook URL'] }}`
- **Input/output connections:** Input from true branch of `If sync enabled`; output to `Store Ghost webhook ID`.
- **Version-specific requirements:** `httpRequest` `typeVersion 4.4`; Ghost Admin API credential must exist.
- **Edge cases or failure types:**
  - Invalid Ghost Admin credentials.
  - Wrong Ghost URL.
  - Duplicate webhook creation if the same event/target pair already exists.
  - Ghost API version/path mismatch on nonstandard Ghost setups.
  - SSL/TLS or network connectivity problems.
- **Sub-workflow reference:** None.

#### Store Ghost webhook ID
- **Type and role:** `dataTable`; saves the created Ghost webhook ID for later deletion.
- **Configuration choices:**
  - Inserts a row into `ghost_webhooks`
  - Maps:
    - `event` = `{{$json.webhooks[0].event}}`
    - `webhook_id` = `{{$json.webhooks[0].id}}`
- **Key expressions or variables used:**
  - `={{ $json.webhooks[0].event }}`
  - `={{ $json.webhooks[0].id }}`
- **Input/output connections:** Input from `Add webhook in Ghost`; output to `Get current workflow`.
- **Version-specific requirements:** `typeVersion 1.1`.
- **Edge cases or failure types:**
  - Fails if `ghost_webhooks` table does not exist.
  - Fails if Ghost API response structure differs.
  - Can create duplicate rows if the workflow repeatedly enables the same event without cleanup.
- **Sub-workflow reference:** None.

---

## 2.5 Ghost Webhook Deletion Path

**Overview:**  
When a sync direction is disabled, this block looks up the previously stored Ghost webhook ID, deletes the webhook in Ghost, then removes the corresponding row from the Data Table.

**Nodes Involved:**
- `Get webhook ID to delete`
- `Delete webhook in Ghost`
- `Forget Ghost webhook ID`

### Node Details

#### Get webhook ID to delete
- **Type and role:** `dataTable`; retrieves stored webhook metadata by event type.
- **Configuration choices:**
  - Operation: `get`
  - Table: `ghost_webhooks`
  - Filter condition:
    - `event = {{$json.type}}`
- **Key expressions or variables used:**
  - `={{ $json.type }}`
- **Input/output connections:** Input from false branch of `If sync enabled`; output to `Delete webhook in Ghost`.
- **Version-specific requirements:** `typeVersion 1.1`.
- **Edge cases or failure types:**
  - If no matching row exists, downstream deletion URL may contain an empty webhook ID.
  - Multiple rows for one event may produce multiple deletions or ambiguous behavior.
  - Requires Data Table access.
- **Sub-workflow reference:** None.

#### Delete webhook in Ghost
- **Type and role:** `httpRequest`; deletes the Ghost webhook by stored ID.
- **Configuration choices:**
  - Method: `DELETE`
  - URL uses:
    - Ghost API URL from form
    - `webhook_id` from Data Table lookup
  - Authentication: `ghostAdminApi`
- **Key expressions or variables used:**
  - `={{ $('Sync Settings Form').item.json['Ghost API URL'] }}/ghost/api/v2/admin/webhooks/{{ $json.webhook_id }}`
- **Input/output connections:** Input from `Get webhook ID to delete`; output to `Forget Ghost webhook ID`.
- **Version-specific requirements:** `httpRequest` `typeVersion 4.4`.
- **Edge cases or failure types:**
  - Missing `webhook_id` leads to invalid endpoint URL.
  - Webhook may already be deleted in Ghost, producing 404-type failures.
  - Invalid credentials or wrong Ghost URL.
- **Sub-workflow reference:** None.

#### Forget Ghost webhook ID
- **Type and role:** `dataTable`; removes the row associated with the disabled event.
- **Configuration choices:**
  - Operation: `deleteRows`
  - Filter:
    - `event = {{ $('Split Out').item.json.type }}`
  - `alwaysOutputData` enabled
- **Key expressions or variables used:**
  - `={{ $('Split Out').item.json.type }}`
- **Input/output connections:** Input from `Delete webhook in Ghost`; output to `Get current workflow`.
- **Version-specific requirements:** `typeVersion 1.1`.
- **Edge cases or failure types:**
  - If multiple rows exist for the same event, all matching rows may be removed.
  - If Ghost deletion fails upstream, this node will not run unless error handling is added.
- **Sub-workflow reference:** None.

---

## 2.6 Workflow Self-Persistence

**Overview:**  
This block reads the current workflow definition, updates the form field default values with the submitted settings, and saves the modified workflow back to n8n. It makes the form “remember” the last chosen configuration.

**Nodes Involved:**
- `Get current workflow`
- `Update default form values to remember webhook status`
- `Update current workflow with sync status`

### Node Details

#### Get current workflow
- **Type and role:** `n8n`; retrieves the currently running workflow definition.
- **Configuration choices:**
  - Operation: `get`
  - Workflow ID: current workflow via `{{$workflow.id}}`
  - `executeOnce` enabled
- **Key expressions or variables used:**
  - `={{$workflow.id}}`
- **Input/output connections:**
  - Input from `Store Ghost webhook ID` and `Forget Ghost webhook ID`
  - Output to `Update default form values to remember webhook status`
- **Version-specific requirements:** `typeVersion 1`; requires n8n credentials with permission to read workflows.
- **Edge cases or failure types:**
  - Requires correct n8n API/auth setup.
  - If self-read is not permitted, the block fails.
  - Multiple incoming executions may race when saving workflow state.
- **Sub-workflow reference:** None.

#### Update default form values to remember webhook status
- **Type and role:** `code`; mutates the retrieved workflow JSON so the form fields reflect current settings.
- **Configuration choices:**
  - Copies `settings.executionOrder`
  - Iterates over `nodes`
  - Finds node named `Sync Settings Form`
  - Updates `defaultValue` for:
    - `Ghost API URL`
    - `Webhook URL`
    - `Ghost Member Added > Added to Mailerlite`
    - `Ghost Member Removed > Removed from Mailerlite`
- **Key expressions or variables used:**
  - `$('Sync Settings Form').first().json[...]`
  - loop over `item.json.nodes`
- **Input/output connections:** Input from `Get current workflow`; output to `Update current workflow with sync status`.
- **Version-specific requirements:** `code` node `typeVersion 2`.
- **Edge cases or failure types:**
  - Strong dependency on exact node name and field labels.
  - If the workflow JSON structure changes in future n8n versions, the code may need adjustment.
  - Self-modifying workflows can be brittle under concurrent edits.
- **Sub-workflow reference:** None.

#### Update current workflow with sync status
- **Type and role:** `n8n`; writes the modified workflow definition back to n8n.
- **Configuration choices:**
  - Operation: `update`
  - Workflow ID: `{{$workflow.id}}`
  - Workflow object: entire current item serialized as JSON string
- **Key expressions or variables used:**
  - `={{$workflow.id}}`
  - `={{ $json.toJsonString() }}`
- **Input/output connections:** Input from `Update default form values to remember webhook status`; no downstream node.
- **Version-specific requirements:** `typeVersion 1`; requires permission to update workflows.
- **Edge cases or failure types:**
  - Invalid workflow object formatting causes update failure.
  - Concurrent executions may overwrite each other.
  - Saving a malformed workflow can break subsequent runs.
- **Sub-workflow reference:** None.

---

## 2.7 Real-Time Ghost Event Listener and MailerLite Sync

**Overview:**  
This execution path is independent from the configuration form. It listens for Ghost webhook payloads and decides whether the member was deleted or added/currently present, then updates MailerLite accordingly.

**Nodes Involved:**
- `Webhook`
- `Check if member deleted`
- `Update a subscriber`
- `Create a subscriber`

### Node Details

#### Webhook
- **Type and role:** `webhook`; public endpoint receiving Ghost webhook calls.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `ghost-mailerlite`
- **Key expressions or variables used:** Downstream nodes read:
  - `$json.body.member.current`
  - `$json.body.member.previous.email`
  - `$json.body.member.current.email`
- **Input/output connections:** Entry point for event-driven sync branch; output to `Check if member deleted`.
- **Version-specific requirements:** `typeVersion 2.1`.
- **Edge cases or failure types:**
  - Endpoint must be reachable by Ghost.
  - If using test URL vs production URL incorrectly, Ghost events may not arrive.
  - Payload shape must match Ghost webhook format expected by downstream nodes.
- **Sub-workflow reference:** None.

#### Check if member deleted
- **Type and role:** `if`; determines whether the Ghost event represents deletion.
- **Configuration choices:** Checks whether `{{$json.body.member.current}}` is empty.
- **Key expressions or variables used:**
  - `={{ $json.body.member.current }}`
- **Input/output connections:**
  - Input from `Webhook`
  - True branch to `Update a subscriber`
  - False branch to `Create a subscriber`
- **Version-specific requirements:** `typeVersion 2.3`.
- **Edge cases or failure types:**
  - Assumes deleted-member payloads have empty `member.current`.
  - Other event structures may be misclassified.
  - Missing `body.member` object causes expression/runtime issues.
- **Sub-workflow reference:** None.

#### Update a subscriber
- **Type and role:** `mailerLite`; unsubscribes a subscriber when a Ghost member is deleted.
- **Configuration choices:**
  - Operation: `update`
  - Subscriber ID set to previous email value
  - Additional field `status = unsubscribed`
- **Key expressions or variables used:**
  - `={{ $json.body.member.previous.email }}`
- **Input/output connections:** Input from true branch of `Check if member deleted`; no downstream node.
- **Version-specific requirements:** `typeVersion 2`; requires MailerLite credentials.
- **Edge cases or failure types:**
  - Uses email as `subscriberId`; this assumes the MailerLite node accepts email for update lookup in this context.
  - If the subscriber does not exist in MailerLite, update may fail.
  - Auth issues or API plan restrictions may apply.
- **Sub-workflow reference:** None.

#### Create a subscriber
- **Type and role:** `mailerLite`; creates a MailerLite subscriber when a Ghost member exists or is added.
- **Configuration choices:**
  - Email from `Webhook` payload current member email
- **Key expressions or variables used:**
  - `={{ $('Webhook').item.json.body.member.current.email }}`
- **Input/output connections:** Input from false branch of `Check if member deleted`; no downstream node.
- **Version-specific requirements:** `typeVersion 2`; requires MailerLite credentials.
- **Edge cases or failure types:**
  - If subscriber already exists, behavior depends on MailerLite node/API semantics.
  - If the incoming event is not truly a creation event but another member update, this may attempt to recreate an existing subscriber.
  - Missing email in payload causes failure.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Visual overview of the workflow |  |  | ## Sync Ghost Members with MailerLite Subscribers in Real-Time<br>This n8n workflow automatically syncs your Ghost CMS members with MailerLite subscribers in real-time using webhooks. When members join or leave your Ghost site, they're automatically added or unsubscribed from MailerLite with no manual exports, imports, or CSV files required. A simple configuration form lets you enable or disable sync settings for member additions and deletions independently. |
| Sticky Note - Configuration | n8n-nodes-base.stickyNote | Visual setup and credential guidance |  |  | ## 📋 Configure Sync Settings<br><br>Before starting, make sure to populate the following nodes with your credentials:<br>➡️ MailerLite (update/create subscriber)<br>➡️ Ghost (add/delete webhook)<br>➡️ n8n (get/update current workflow)<br><br>Use this form to configure your sync settings:<br>✅ Set your Ghost API URL<br>✅ Set the webhook URL (copy from Webhook node below)<br>✅ Enable/disable member addition sync<br>✅ Enable/disable member deletion sync<br><br>The workflow will automatically create webhooks in Ghost and remember your settings. You may need to refresh the page to get the current sync status displayed in the form. |
| Sticky Note - Webhook Management | n8n-nodes-base.stickyNote | Visual description of Ghost webhook lifecycle management |  |  | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Sticky Note - Sync Listener | n8n-nodes-base.stickyNote | Visual description of incoming event handling |  |  | ## 🎯 Webhook Listener to Sync in Real-Time<br><br>This webhook receives events from Ghost when members are added or deleted.<br><br>**Member Added** → Creates subscriber in MailerLite<br>**Member Deleted** → Unsubscribes from MailerLite<br><br>Ensure this webhook URL is added to your Ghost configuration form above. |
| Sticky Note - State Persistence | n8n-nodes-base.stickyNote | Visual description of self-updating workflow behavior |  |  | ## 💾 Keep Track of Current Sync Status<br><br>After webhook changes, the workflow updates itself to remember your configuration:<br><br>1️⃣ Fetches current workflow<br>2️⃣ Updates form default values with your settings and current sync status<br>3️⃣ Saves the updated workflow<br><br>This makes sure the latest status of your syncs is displayed in the Sync Settings Form. You may need to refresh your browser page if you want to see the settings update immediately after changes are made. |
| Create ghost_webhooks table | n8n-nodes-base.dataTable | Creates persistence table for Ghost webhook IDs | Check if ghost_webhooks table exists | Merge | ## 📋 Configure Sync Settings<br><br>Before starting, make sure to populate the following nodes with your credentials:<br>➡️ MailerLite (update/create subscriber)<br>➡️ Ghost (add/delete webhook)<br>➡️ n8n (get/update current workflow)<br><br>Use this form to configure your sync settings:<br>✅ Set your Ghost API URL<br>✅ Set the webhook URL (copy from Webhook node below)<br>✅ Enable/disable member addition sync<br>✅ Enable/disable member deletion sync<br><br>The workflow will automatically create webhooks in Ghost and remember your settings. You may need to refresh the page to get the current sync status displayed in the form. |
| Webhook | n8n-nodes-base.webhook | Receives Ghost member event callbacks |  | Check if member deleted | ## 🎯 Webhook Listener to Sync in Real-Time<br><br>This webhook receives events from Ghost when members are added or deleted.<br><br>**Member Added** → Creates subscriber in MailerLite<br>**Member Deleted** → Unsubscribes from MailerLite<br><br>Ensure this webhook URL is added to your Ghost configuration form above. |
| Sync Settings Form | n8n-nodes-base.formTrigger | Manual configuration entry point |  | List data tables | ## 📋 Configure Sync Settings<br><br>Before starting, make sure to populate the following nodes with your credentials:<br>➡️ MailerLite (update/create subscriber)<br>➡️ Ghost (add/delete webhook)<br>➡️ n8n (get/update current workflow)<br><br>Use this form to configure your sync settings:<br>✅ Set your Ghost API URL<br>✅ Set the webhook URL (copy from Webhook node below)<br>✅ Enable/disable member addition sync<br>✅ Enable/disable member deletion sync<br><br>The workflow will automatically create webhooks in Ghost and remember your settings. You may need to refresh the page to get the current sync status displayed in the form. |
| List data tables | n8n-nodes-base.dataTable | Lists available Data Tables | Sync Settings Form | Check if ghost_webhooks table exists | ## 📋 Configure Sync Settings<br><br>Before starting, make sure to populate the following nodes with your credentials:<br>➡️ MailerLite (update/create subscriber)<br>➡️ Ghost (add/delete webhook)<br>➡️ n8n (get/update current workflow)<br><br>Use this form to configure your sync settings:<br>✅ Set your Ghost API URL<br>✅ Set the webhook URL (copy from Webhook node below)<br>✅ Enable/disable member addition sync<br>✅ Enable/disable member deletion sync<br><br>The workflow will automatically create webhooks in Ghost and remember your settings. You may need to refresh the page to get the current sync status displayed in the form. |
| Merge | n8n-nodes-base.merge | Rejoins initialization branches | Check if ghost_webhooks table exists; Create ghost_webhooks table | Identity syncs to be enabled/disabled | ## 📋 Configure Sync Settings<br><br>Before starting, make sure to populate the following nodes with your credentials:<br>➡️ MailerLite (update/create subscriber)<br>➡️ Ghost (add/delete webhook)<br>➡️ n8n (get/update current workflow)<br><br>Use this form to configure your sync settings:<br>✅ Set your Ghost API URL<br>✅ Set the webhook URL (copy from Webhook node below)<br>✅ Enable/disable member addition sync<br>✅ Enable/disable member deletion sync<br><br>The workflow will automatically create webhooks in Ghost and remember your settings. You may need to refresh the page to get the current sync status displayed in the form. |
| Split Out | n8n-nodes-base.splitOut | Emits one desired webhook action per item | Identity syncs to be enabled/disabled | If sync enabled | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Check if member deleted | n8n-nodes-base.if | Distinguishes deletion vs non-deletion events | Webhook | Update a subscriber; Create a subscriber | ## 🎯 Webhook Listener to Sync in Real-Time<br><br>This webhook receives events from Ghost when members are added or deleted.<br><br>**Member Added** → Creates subscriber in MailerLite<br>**Member Deleted** → Unsubscribes from MailerLite<br><br>Ensure this webhook URL is added to your Ghost configuration form above. |
| Update a subscriber | n8n-nodes-base.mailerLite | Unsubscribes a MailerLite subscriber after Ghost deletion | Check if member deleted |  | ## 🎯 Webhook Listener to Sync in Real-Time<br><br>This webhook receives events from Ghost when members are added or deleted.<br><br>**Member Added** → Creates subscriber in MailerLite<br>**Member Deleted** → Unsubscribes from MailerLite<br><br>Ensure this webhook URL is added to your Ghost configuration form above. |
| Create a subscriber | n8n-nodes-base.mailerLite | Creates a MailerLite subscriber from Ghost member data | Check if member deleted |  | ## 🎯 Webhook Listener to Sync in Real-Time<br><br>This webhook receives events from Ghost when members are added or deleted.<br><br>**Member Added** → Creates subscriber in MailerLite<br>**Member Deleted** → Unsubscribes from MailerLite<br><br>Ensure this webhook URL is added to your Ghost configuration form above. |
| Get webhook ID to delete | n8n-nodes-base.dataTable | Looks up stored Ghost webhook ID by event | If sync enabled | Delete webhook in Ghost | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Delete webhook in Ghost | n8n-nodes-base.httpRequest | Deletes a Ghost webhook through Admin API | Get webhook ID to delete | Forget Ghost webhook ID | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Add webhook in Ghost | n8n-nodes-base.httpRequest | Creates a Ghost webhook through Admin API | If sync enabled | Store Ghost webhook ID | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Store Ghost webhook ID | n8n-nodes-base.dataTable | Persists newly created Ghost webhook IDs | Add webhook in Ghost | Get current workflow | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Forget Ghost webhook ID | n8n-nodes-base.dataTable | Removes stored Ghost webhook row after deletion | Delete webhook in Ghost | Get current workflow | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Update default form values to remember webhook status | n8n-nodes-base.code | Mutates workflow JSON so the form remembers last settings | Get current workflow | Update current workflow with sync status | ## 💾 Keep Track of Current Sync Status<br><br>After webhook changes, the workflow updates itself to remember your configuration:<br><br>1️⃣ Fetches current workflow<br>2️⃣ Updates form default values with your settings and current sync status<br>3️⃣ Saves the updated workflow<br><br>This makes sure the latest status of your syncs is displayed in the Sync Settings Form. You may need to refresh your browser page if you want to see the settings update immediately after changes are made. |
| Get current workflow | n8n-nodes-base.n8n | Reads current workflow definition from n8n | Store Ghost webhook ID; Forget Ghost webhook ID | Update default form values to remember webhook status | ## 💾 Keep Track of Current Sync Status<br><br>After webhook changes, the workflow updates itself to remember your configuration:<br><br>1️⃣ Fetches current workflow<br>2️⃣ Updates form default values with your settings and current sync status<br>3️⃣ Saves the updated workflow<br><br>This makes sure the latest status of your syncs is displayed in the Sync Settings Form. You may need to refresh your browser page if you want to see the settings update immediately after changes are made. |
| Update current workflow with sync status | n8n-nodes-base.n8n | Saves modified workflow definition back to n8n | Update default form values to remember webhook status |  | ## 💾 Keep Track of Current Sync Status<br><br>After webhook changes, the workflow updates itself to remember your configuration:<br><br>1️⃣ Fetches current workflow<br>2️⃣ Updates form default values with your settings and current sync status<br>3️⃣ Saves the updated workflow<br><br>This makes sure the latest status of your syncs is displayed in the Sync Settings Form. You may need to refresh your browser page if you want to see the settings update immediately after changes are made. |
| Check if ghost_webhooks table exists | n8n-nodes-base.if | Determines whether initialization should create the data table | List data tables | Merge; Create ghost_webhooks table | ## 📋 Configure Sync Settings<br><br>Before starting, make sure to populate the following nodes with your credentials:<br>➡️ MailerLite (update/create subscriber)<br>➡️ Ghost (add/delete webhook)<br>➡️ n8n (get/update current workflow)<br><br>Use this form to configure your sync settings:<br>✅ Set your Ghost API URL<br>✅ Set the webhook URL (copy from Webhook node below)<br>✅ Enable/disable member addition sync<br>✅ Enable/disable member deletion sync<br><br>The workflow will automatically create webhooks in Ghost and remember your settings. You may need to refresh the page to get the current sync status displayed in the form. |
| If sync enabled | n8n-nodes-base.if | Routes each desired event toward create or delete logic | Split Out | Add webhook in Ghost; Get webhook ID to delete | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |
| Identity syncs to be enabled/disabled | n8n-nodes-base.code | Builds normalized desired sync actions from form selections | Merge | Split Out | ## 🔗 Webhook Management<br><br>This section automatically manages Ghost webhooks based on your form settings:<br><br>When **enabled**: Creates webhook in Ghost → Stores webhook ID in data table<br>When **disabled**: Retrieves webhook ID → Deletes webhook from Ghost → Removes from data table<br><br>Webhook IDs are stored so they can be properly removed later when unsyncing. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Sticky Note** for the overview.
   - Title/content should describe that Ghost members are synced to MailerLite in real time.
   - This is optional for execution but useful for maintainability.

3. **Add a Form Trigger node** named `Sync Settings Form`.
   - Set **Form Title** to `Sync Settings Form`.
   - Set **Button Label** to `Update Sync`.
   - Add four form fields:
     1. `Ghost API URL`
     2. `Webhook URL`
     3. Radio field `Ghost Member Added > Added to Mailerlite`
        - Options: `Enabled`, `Disabled`
        - Default: `Disabled`
     4. Radio field `Ghost Member Removed > Removed from Mailerlite`
        - Options: `Enabled`, `Disabled`
        - Default: `Disabled`

4. **Add a Data Table node** named `List data tables`.
   - Resource: `table`
   - Operation: list tables
   - Enable `Always Output Data`.

5. **Connect** `Sync Settings Form` → `List data tables`.

6. **Add an IF node** named `Check if ghost_webhooks table exists`.
   - Condition type: object
   - Check whether `{{$json}}` is **not empty**.
   - Enable `Execute Once` if you want to mirror the original behavior.

7. **Connect** `List data tables` → `Check if ghost_webhooks table exists`.

8. **Add a Data Table node** named `Create ghost_webhooks table`.
   - Resource: `table`
   - Operation: `create`
   - Table name: `ghost_webhooks`
   - Add columns:
     - `webhook_id`
     - `event`

9. **Connect** the **false** output of `Check if ghost_webhooks table exists` → `Create ghost_webhooks table`.

10. **Add a Merge node** named `Merge`.
    - Leave default configuration unless your n8n version requires an explicit merge mode.

11. **Connect**:
    - `Check if ghost_webhooks table exists` true output → `Merge` input 1
    - `Create ghost_webhooks table` → `Merge` input 2

12. **Add a Code node** named `Identity syncs to be enabled/disabled`.
    - Paste logic equivalent to:
      - create `item.json.webhooks = []`
      - push a config object for `member.added`
      - push a config object for `member.deleted`
      - set `action` to `enabled` or `disabled` based on form radio values
    - The key field names must remain exactly:
      - `Ghost Member Added > Added to Mailerlite`
      - `Ghost Member Removed > Removed from Mailerlite`

13. **Connect** `Merge` → `Identity syncs to be enabled/disabled`.

14. **Add a Split Out node** named `Split Out`.
    - Field to split out: `webhooks`

15. **Connect** `Identity syncs to be enabled/disabled` → `Split Out`.

16. **Add an IF node** named `If sync enabled`.
    - Condition: string equals
    - Left value: `{{$json.action}}`
    - Right value: `enabled`

17. **Connect** `Split Out` → `If sync enabled`.

18. **Create Ghost Admin API credentials** in n8n.
    - Credential type: `Ghost Admin API` or the credential type required by the HTTP Request node.
    - Use your Ghost Admin API key and correct Ghost Admin domain/base URL as needed by n8n.

19. **Add an HTTP Request node** named `Add webhook in Ghost`.
    - Method: `POST`
    - URL:
      `{{ $('Sync Settings Form').item.json['Ghost API URL'] }}/ghost/api/v2/admin/webhooks`
    - Authentication: predefined credential
    - Credential type: `ghostAdminApi`
    - Send body: enabled
    - Body type: JSON
    - JSON body:
      - root object with `webhooks` array
      - one object containing:
        - `event` = current split item type
        - `target_url` = submitted `Webhook URL`

20. **Connect** the **true** output of `If sync enabled` → `Add webhook in Ghost`.

21. **Add a Data Table node** named `Store Ghost webhook ID`.
    - Target table: `ghost_webhooks`
    - Insert row
    - Map:
      - `event` from created webhook response event
      - `webhook_id` from created webhook response ID

22. **Connect** `Add webhook in Ghost` → `Store Ghost webhook ID`.

23. **Add a Data Table node** named `Get webhook ID to delete`.
    - Operation: `get`
    - Table: `ghost_webhooks`
    - Filter rows where:
      - `event = {{$json.type}}`

24. **Connect** the **false** output of `If sync enabled` → `Get webhook ID to delete`.

25. **Add an HTTP Request node** named `Delete webhook in Ghost`.
    - Method: `DELETE`
    - URL:
      `{{ $('Sync Settings Form').item.json['Ghost API URL'] }}/ghost/api/v2/admin/webhooks/{{ $json.webhook_id }}`
    - Authentication: predefined credential
    - Credential type: `ghostAdminApi`

26. **Connect** `Get webhook ID to delete` → `Delete webhook in Ghost`.

27. **Add a Data Table node** named `Forget Ghost webhook ID`.
    - Operation: `deleteRows`
    - Table: `ghost_webhooks`
    - Filter rows where:
      - `event = {{ $('Split Out').item.json.type }}`

28. **Connect** `Delete webhook in Ghost` → `Forget Ghost webhook ID`.

29. **Create n8n credentials** for the n8n node.
    - Use credentials that can **get** and **update** workflows.
    - On self-hosted n8n, this often means an API key or internal auth with sufficient permission scope.

30. **Add an n8n node** named `Get current workflow`.
    - Operation: `get`
    - Workflow ID: `{{$workflow.id}}`
    - Enable `Execute Once`.

31. **Connect** both:
    - `Store Ghost webhook ID` → `Get current workflow`
    - `Forget Ghost webhook ID` → `Get current workflow`

32. **Add a Code node** named `Update default form values to remember webhook status`.
    - Implement logic that:
      - loops through the returned workflow object,
      - finds the node named `Sync Settings Form`,
      - updates the default value of:
        - `Ghost API URL`
        - `Webhook URL`
        - `Ghost Member Added > Added to Mailerlite`
        - `Ghost Member Removed > Removed from Mailerlite`
      - uses values from the current form submission.
    - Preserve required workflow settings such as `settings.executionOrder`.

33. **Connect** `Get current workflow` → `Update default form values to remember webhook status`.

34. **Add an n8n node** named `Update current workflow with sync status`.
    - Operation: `update`
    - Workflow ID: `{{$workflow.id}}`
    - Workflow object: serialize the modified workflow JSON into a string, for example with `{{$json.toJsonString()}}`

35. **Connect** `Update default form values to remember webhook status` → `Update current workflow with sync status`.

36. **Add a Webhook node** named `Webhook`.
    - HTTP Method: `POST`
    - Path: `ghost-mailerlite`
    - After saving/activating the workflow, copy the production URL and use it in the form field `Webhook URL`.

37. **Add an IF node** named `Check if member deleted`.
    - Condition: object is empty
    - Left value: `{{$json.body.member.current}}`
    - This should route:
      - true = deleted
      - false = current member exists

38. **Connect** `Webhook` → `Check if member deleted`.

39. **Create MailerLite credentials** in n8n.
    - Use the MailerLite credential type required by the MailerLite node.
    - Ensure the account can create and update subscribers.

40. **Add a MailerLite node** named `Update a subscriber`.
    - Operation: `update`
    - Subscriber identifier:
      `{{$json.body.member.previous.email}}`
    - Additional field:
      - `status = unsubscribed`

41. **Connect** the **true** output of `Check if member deleted` → `Update a subscriber`.

42. **Add a MailerLite node** named `Create a subscriber`.
    - Operation: create subscriber
    - Email:
      `{{ $('Webhook').item.json.body.member.current.email }}`

43. **Connect** the **false** output of `Check if member deleted` → `Create a subscriber`.

44. **Add the remaining Sticky Notes** to mirror the original layout:
    - Configuration note
    - Webhook management note
    - Sync listener note
    - State persistence note

45. **Save the workflow** and then **activate it**.

46. **Test the configuration branch**:
    - Open the Form Trigger URL.
    - Enter:
      - Ghost API URL, e.g. `https://your-ghost-site.com`
      - Webhook URL from the `Webhook` node production endpoint
      - enable or disable each sync direction
    - Submit the form.

47. **Verify Ghost webhooks**:
    - In Ghost Admin or via API, confirm that `member.added` and/or `member.deleted` webhooks were created or deleted as expected.

48. **Verify Data Table content**:
    - Confirm that table `ghost_webhooks` contains rows with:
      - `event`
      - `webhook_id`

49. **Verify self-persistence**:
    - Refresh the form page.
    - Confirm default values now reflect the last submitted values.

50. **Test runtime events**:
    - Add a member in Ghost and confirm a MailerLite subscriber is created.
    - Delete a member in Ghost and confirm the MailerLite subscriber is unsubscribed.

## Credential setup summary

- **Ghost Admin API credential**
  - Used by:
    - `Add webhook in Ghost`
    - `Delete webhook in Ghost`

- **MailerLite credential**
  - Used by:
    - `Create a subscriber`
    - `Update a subscriber`

- **n8n credential**
  - Used by:
    - `Get current workflow`
    - `Update current workflow with sync status`

## Important rebuild constraints

- Keep these node names unchanged if you want the expressions to work exactly:
  - `Sync Settings Form`
  - `Webhook`
  - `Split Out`

- Keep these form labels unchanged unless you also update every dependent expression and code block:
  - `Ghost API URL`
  - `Webhook URL`
  - `Ghost Member Added > Added to Mailerlite`
  - `Ghost Member Removed > Removed from Mailerlite`

- Keep the Data Table name as:
  - `ghost_webhooks`

- Keep the webhook path as:
  - `ghost-mailerlite`
  unless you also update the URL used in Ghost.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is self-modifying: it updates its own form defaults after sync settings change. | Relevant to `Get current workflow`, `Update default form values to remember webhook status`, and `Update current workflow with sync status` |
| The table-existence check is weak: it checks for any non-empty Data Table listing, not specifically a table named `ghost_webhooks`. In some environments, this should be improved. | Recommended maintenance improvement |
| The MailerLite create path may also run for non-delete Ghost events other than true member creation if Ghost sends broader member lifecycle payloads. | Relevant to webhook payload design |
| The form may require a browser refresh before updated default values become visible. | Mentioned in workflow notes |
| The submitted Webhook URL should be the production URL of the `Webhook` node, not a temporary test URL. | Relevant to Ghost webhook delivery |