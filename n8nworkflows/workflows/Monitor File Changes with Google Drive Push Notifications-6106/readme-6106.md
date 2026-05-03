Monitor File Changes with Google Drive Push Notifications

https://n8nworkflows.xyz/workflows/monitor-file-changes-with-google-drive-push-notifications-6106


# Monitor File Changes with Google Drive Push Notifications

### 1. Workflow Overview

This workflow enables monitoring of Google Drive file changes using Google Drive Push Notifications, which are webhook-based notifications triggered by Google Drive change events. It replaces inefficient polling methods with an event-driven approach to efficiently detect file changes, especially focusing on PDF files within a specified drive.

The workflow is logically divided into the following blocks:

- **1.1 Initialization & Registration**: Sets essential variables and registers a Google Drive notification channel (webhook) to start receiving push notifications about file changes.
- **1.2 Scheduled Renewal**: Periodically renews the notification channel to keep it active, as Google Drive channels expire after one week.
- **1.3 Webhook Reception & Validation**: Listens for incoming push notifications, validates them, and filters out invalid or irrelevant requests.
- **1.4 Change Event Retrieval & Filtering**: Fetches the list of file changes since the last notification, filters for relevant file changes (specifically non-removed PDF files), and obtains detailed file metadata.
- **1.5 Token Management**: Updates and persists the page token to ensure continuous, incremental change tracking.
- **1.6 Post-Processing**: Placeholder node to handle valid file changes (e.g., further automation or processing).

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Registration
- **Overview**: Prepares and registers a Google Drive notification channel (webhook) for receiving push notifications on file changes within a specified drive.
- **Nodes Involved**: Set Variables, Get StartPageToken, Register Webhook, Save Registration to WorkflowStaticData, Stop Any Existing Notifications, Get Registration, Register Notification Channel, First Run? Click Here!, Subworkflow Trigger
- **Node Details**:
  - **Set Variables**
    - Type: Set node
    - Role: Stores configurable parameters such as `driveId`, `channelId`, `channelToken`, and computes the webhook URL dynamically based on execution context.
    - Key Variables: 
      - `driveId`: Identifier for the target Google Drive (must be replaced by user).
      - `channelId`: Custom channel name for notification channel.
      - `channelToken`: Authentication token for verification of notifications.
      - `webhookUrl`: URL of the webhook to receive notifications.
    - Input: Triggered by Schedule Trigger or Subworkflow Trigger
    - Output: Passes variables to downstream nodes.
    - Edge Cases: User must replace placeholders; if invalid or missing, registration will fail.
  
  - **Get StartPageToken**
    - Type: HTTP Request
    - Role: Requests the current start page token from Google Drive API to bookmark the change list cursor.
    - Configuration: Uses OAuth2 Google Drive credentials; queries with `supportsAllDrives=true` and user-defined `driveId`.
    - Input: From Set Variables
    - Output: Provides a startPageToken for the registration.
    - Edge Cases: API failures, auth errors; invalid `driveId` may cause errors.
  
  - **Register Webhook**
    - Type: HTTP Request (POST)
    - Role: Registers the notification channel with Google Drive API using the startPageToken.
    - Key Configurations:
      - Body contains `id` (channelId), `type` (web_hook), `address` (webhookUrl), `token` (channelToken), and `expiration` (7 days from now).
      - Query parameters include `pageToken` (startPageToken), `supportsAllDrives=true`, `includeItemsFromAllDrives=true`, and `driveId`.
    - Input: From Get StartPageToken
    - Output: Registration response with resourceId and other metadata.
    - Edge Cases: Token expiry, invalid webhook URL, network issues.
  
  - **Save Registration to WorkflowStaticData**
    - Type: Code
    - Role: Saves registration response and startPageToken persistently in `workflowStaticData` for future reference.
    - Input: From Register Webhook
    - Output: Updates global static data storage.
    - Edge Cases: Requires production mode to persist data.
  
  - **Stop Any Existing Notifications**
    - Type: HTTP Request (POST)
    - Role: Stops any pre-existing notification channels to avoid duplicates before re-registration.
    - Input: From Get Registration (loads from static data)
    - Output: Passes to Get StartPageToken for fresh registration.
    - Edge Cases: Continues on error to avoid blocking workflow.
  
  - **Get Registration**
    - Type: Code
    - Role: Retrieves existing registration data from `workflowStaticData`.
    - Input: From Set Variables
    - Output: Passes registration info for stopping existing channels.
  
  - **Register Notification Channel**
    - Type: Execute Workflow
    - Role: Triggers this same workflow to perform the registration process for first-time setup.
    - Input: From Manual Trigger "First Run? Click Here!"
    - Output: Initiates Set Variables node.
  
  - **First Run? Click Here!**
    - Type: Manual Trigger
    - Role: Entry point for manual start to register the notification channel initially.
  
  - **Subworkflow Trigger**
    - Type: Execute Workflow Trigger
    - Role: Alternative trigger to initiate Set Variables and registration.

---

#### 1.2 Scheduled Renewal
- **Overview**: Keeps the notification channel active by renewing registration every 6 days before expiration.
- **Nodes Involved**: Schedule Trigger, Set Variables, Get Registration, Stop Any Existing Notifications, Get StartPageToken, Register Webhook, Save Registration to WorkflowStaticData
- **Node Details**:
  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Runs the registration renewal workflow every 6 days at 6 AM.
    - Input: None
    - Output: Triggers Set Variables
  - The remaining nodes are as in 1.1 Initialization, ensuring the channel is renewed.

---

#### 1.3 Webhook Reception & Validation
- **Overview**: Webhook node receives push notifications from Google Drive, and the workflow validates incoming requests for authenticity and relevance.
- **Nodes Involved**: Webhook, Get Registration From WorkflowStaticData, Is Valid Request?, Invalid Request, Get Changes List
- **Node Details**:
  - **Webhook**
    - Type: Webhook Trigger
    - Role: Receives POST requests from Google Drive push notifications.
    - Configuration: HTTP POST method, path unique to the workflow instance.
    - Input: External Google Drive push notification.
    - Output: Passes request data downstream.
    - Edge Cases: Unauthorized requests, malformed payloads.
  
  - **Get Registration From WorkflowStaticData**
    - Type: Code
    - Role: Retrieves stored registration data from `workflowStaticData` to validate incoming requests.
    - Input: From Webhook
    - Output: Passes data for validation.
  
  - **Is Valid Request?**
    - Type: If
    - Role: Checks that the notification headers match the expected channel ID, token, and resource state is "change".
    - Conditions:
      - `x-goog-channel-id` matches stored registration id.
      - `x-goog-channel-token` matches stored token.
      - `x-goog-resource-state` equals "change".
    - Output:
      - True: Continue to Get Changes List.
      - False: Go to Invalid Request.
    - Edge Cases: Missing headers, invalid tokens, replay attacks.
  
  - **Invalid Request**
    - Type: No Operation
    - Role: Placeholder for handling invalid requests (e.g., logging or ignoring).
    - Input: From Is Valid Request? (false branch)

---

#### 1.4 Change Event Retrieval & Filtering
- **Overview**: Retrieves the list of changes from Google Drive since the last known token, splits the events, filters for PDF files that were changed and not removed, and retrieves full file metadata.
- **Nodes Involved**: Get Changes List, Save NextPageToken to WorkflowStaticData, Change Events to Items, Filter Changed Events, Get Files Details, Do Something With These Files!
- **Node Details**:
  - **Get Changes List**
    - Type: HTTP Request
    - Role: Calls Google Drive Changes API with the last saved pageToken to get new change events.
    - Query Parameters:
      - `pageToken`: lastPageToken from `workflowStaticData`
      - `supportsAllDrives=true`, `includeItemsFromAllDrives=true`, `driveId`
    - Input: From Is Valid Request? (true branch)
    - Output: JSON with change events and next page token.
    - Edge Cases: Token expiration, API errors.
  
  - **Save NextPageToken to WorkflowStaticData**
    - Type: Code
    - Role: Updates `workflowStaticData` with the new page token (`newStartPageToken` or `nextPageToken`) for future incremental queries.
    - Input: From Get Changes List
  
  - **Change Events to Items**
    - Type: SplitOut
    - Role: Splits the array of change events into individual items for filtering.
    - Input: From Get Changes List
  
  - **Filter Changed Events**
    - Type: Filter
    - Role: Filters change events to:
      - Not removed (`removed` is false)
      - Type equals "file"
      - Change type equals "file"
      - File MIME type equals "application/pdf" (PDF files)
    - Input: From Change Events to Items
  
  - **Get Files Details**
    - Type: HTTP Request
    - Role: Fetches detailed metadata for each filtered file by ID from the Drive API.
    - Query Parameters:
      - Fields: `id,kind,name,createdTime,modifiedTime,mimeType,trashed,driveId,parents`
      - `supportsAllDrives=true`
    - Input: From Filter Changed Events
    - Output: File metadata for further processing
  
  - **Do Something With These Files!**
    - Type: No Operation
    - Role: Placeholder for downstream processing of the file metadata (e.g., notifications, processing workflows).
    - Input: From Get Files Details

---

#### 1.5 Token Management
- **Overview**: Ensures continuous tracking of changes by saving and updating the page token in the global `workflowStaticData`.
- **Nodes Involved**: Save Registration to WorkflowStaticData, Save NextPageToken to WorkflowStaticData
- **Node Details**:
  - Both nodes are Code nodes that persist tokens in `workflowStaticData` with necessary logic to handle first-time and ongoing updates.
  - Edge Cases: Tokens must be stored persistently; workflow must run in production mode for `workflowStaticData` to persist.

---

#### 1.6 Post-Processing
- **Overview**: Placeholder node for custom logic to handle changed files after filtering and metadata retrieval.
- **Nodes Involved**: Do Something With These Files!
- **Node Details**:
  - No Operation node intended to be replaced or extended by users to implement custom automations.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                                  | Input Node(s)                          | Output Node(s)                      | Sticky Note                                                                                                                           |
|-----------------------------------|----------------------------|-------------------------------------------------|--------------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Set Variables                     | Set                        | Define variables: driveId, channelId, channelToken, webhookUrl | Schedule Trigger, Subworkflow Trigger | Get Registration                   | ⚠️ Edit me! Add your details here and the template will do the rest.                                                                  |
| Get StartPageToken                | HTTP Request               | Get start page token from Google Drive API      | Stop Any Existing Notifications, Register Webhook | Register Webhook                   | Renewing the Notification Channel: Channels expire weekly; renew every 6 days.                                                        |
| Register Webhook                 | HTTP Request               | Register Google Drive webhook notification channel | Get StartPageToken                   | Save Registration to WorkflowStaticData | 1. Register a Google Drive Notification Channel (Webhook). Link: https://developers.google.com/workspace/drive/api/guides/push         |
| Save Registration to WorkflowStaticData | Code                      | Save registration data and startPageToken        | Register Webhook                     | -                                  | 3. Update LastPageToken For Next Push Notification. Link: https://docs.n8n.io/code/cookbook/builtin/get-workflow-static-data/          |
| Stop Any Existing Notifications  | HTTP Request               | Stop any existing notification channel           | Get Registration                    | Get StartPageToken                 |                                                                                                                                        |
| Get Registration                 | Code                      | Retrieve registration data from workflowStaticData | Set Variables                      | Stop Any Existing Notifications    |                                                                                                                                        |
| Register Notification Channel    | Execute Workflow           | Trigger registration workflow                     | First Run? Click Here!               | Set Variables                     |                                                                                                                                        |
| First Run? Click Here!           | Manual Trigger             | Manual start for initial registration             | -                                  | Register Notification Channel      |                                                                                                                                        |
| Subworkflow Trigger             | Execute Workflow Trigger   | Alternative trigger for registration               | -                                  | Set Variables                     |                                                                                                                                        |
| Schedule Trigger                | Schedule Trigger           | Periodic trigger to renew notification channel    | -                                  | Set Variables                     | Renewing the Notification Channel: Channels expire weekly; renew every 6 days.                                                        |
| Webhook                        | Webhook                   | Receive push notifications from Google Drive      | External Google Drive               | Get Registration From WorkflowStaticData | 2. Listening For Push Notifications. Link: https://developers.google.com/workspace/drive/api/guides/push#receive-notifications          |
| Get Registration From WorkflowStaticData | Code                      | Retrieve registration data for validation          | Webhook                           | Is Valid Request?                 |                                                                                                                                        |
| Is Valid Request?              | If                        | Validate authenticity and relevance of notification | Get Registration From WorkflowStaticData | Get Changes List, Invalid Request |                                                                                                                                        |
| Invalid Request               | No Operation              | Handle invalid or unauthorized notification       | Is Valid Request? (false)           | -                                  |                                                                                                                                        |
| Get Changes List             | HTTP Request               | Fetch list of change events from Google Drive API | Is Valid Request? (true)             | Save NextPageToken to WorkflowStaticData, Change Events to Items | 4. Post-Filter the Changes List.                                                                                                      |
| Save NextPageToken to WorkflowStaticData | Code                      | Save updated page token for incremental sync       | Get Changes List                   | -                                  | 3. Update LastPageToken For Next Push Notification.                                                                                    |
| Change Events to Items       | SplitOut                  | Split changes array into individual events         | Get Changes List                   | Filter Changed Events             | 4. Post-Filter the Changes List.                                                                                                      |
| Filter Changed Events        | Filter                    | Filter only relevant file change events (PDF files, not removed) | Change Events to Items             | Get Files Details                | 4. Post-Filter the Changes List.                                                                                                      |
| Get Files Details            | HTTP Request               | Retrieve detailed metadata for filtered files      | Filter Changed Events              | Do Something With These Files!     | 4. Post-Filter the Changes List.                                                                                                      |
| Do Something With These Files! | No Operation              | Placeholder for downstream processing              | Get Files Details                  | -                                  |                                                                                                                                        |
| Sticky Note                   | Sticky Note               | Explanation on Google Drive notification channel    | -                                  | -                                  | See detailed notes in section 5 below.                                                                                                |
| Sticky Note2                  | Sticky Note               | Notes on listening and activating Webhook trigger  | -                                  | -                                  |                                                                                                                                        |
| Sticky Note3                  | Sticky Note               | Notes on page token management                       | -                                  | -                                  |                                                                                                                                        |
| Sticky Note4                  | Sticky Note               | Notes on filtering changes and file metadata        | -                                  | -                                  |                                                                                                                                        |
| Sticky Note5                  | Sticky Note               | Comprehensive overview of workflow and usage        | -                                  | -                                  | https://discord.com/invite/XPKeKXeB7d (Discord), https://community.n8n.io/ (Forum)                                                     |
| Sticky Note6                  | Sticky Note               | Notes on renewing the notification channel          | -                                  | -                                  |                                                                                                                                        |
| Sticky Note7                  | Sticky Note               | Placeholder for user to edit variables               | -                                  | -                                  |                                                                                                                                        |
| Sticky Note8                  | Sticky Note               | Banner image                                          | -                                  | -                                  |                                                                                                                                        |
| cbaee96d-e408-4c02-b05c-14a8d5348d56 (Sticky Note1) | Sticky Note               | Warning about WorkflowStaticData only working in production mode | -                                  | -                                  |                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: "First Run? Click Here!"
   - Purpose: Initiate first-time registration process.

2. **Create Execute Workflow Node**
   - Type: Execute Workflow
   - Name: "Register Notification Channel"
   - Purpose: Calls the same workflow to perform registration.
   - Connect Manual Trigger to this node.

3. **Create Set Node**
   - Type: Set
   - Name: "Set Variables"
   - Parameters:
     - `driveId`: Replace with your target Google Drive ID.
     - `channelId`: Custom channel name (string).
     - `channelToken`: Custom token string for webhook auth.
     - `webhookUrl`: Expression to compute webhook URL dynamically:
       ```
       ={{ 'https://' + $execution.resumeUrl.extractDomain() + '/webhook/' +$('Webhook').params.path }}
       ```
   - Connect "Register Notification Channel" (execute workflow) and Schedule Trigger to this node.

4. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Name: "Schedule Trigger"
   - Parameters:
     - Interval: Every 6 days at 06:00
   - Connect to "Set Variables" for scheduled renewal.

5. **Create Code Node**
   - Type: Code
   - Name: "Get Registration"
   - Purpose: Retrieve existing registration data from global `workflowStaticData`.
   - Code:
     ```javascript
     return $getWorkflowStaticData('global') ?? {};
     ```
   - Connect "Set Variables" to this node.

6. **Create HTTP Request Node**
   - Type: HTTP Request
   - Name: "Stop Any Existing Notifications"
   - Method: POST
   - URL: `https://www.googleapis.com/drive/v3/channels/stop`
   - Body Parameters:
     - `id`: `={{ $json.registration?.id ?? $('Set Variables').first().json.channelId }}`
     - `resourceId`: `={{ $json.registration?.resourceId }}`
     - `token`: `={{ $json.registration?.token ?? $('Set Variables').first().json.channelToken }}`
   - Authentication: Use Google Drive OAuth2 credentials.
   - Connect "Get Registration" to this node.
   - On error: Continue regular output.

7. **Create HTTP Request Node**
   - Type: HTTP Request
   - Name: "Get StartPageToken"
   - Method: GET
   - URL: `https://www.googleapis.com/drive/v3/changes/startPageToken`
   - Query Parameters:
     - `supportsAllDrives`: `true`
     - `driveId`: `={{ $('Set Variables').first().json.driveId }}`
   - Authentication: Google Drive OAuth2
   - Connect "Stop Any Existing Notifications" to this node.

8. **Create HTTP Request Node**
   - Type: HTTP Request
   - Name: "Register Webhook"
   - Method: POST
   - URL: `https://www.googleapis.com/drive/v3/changes/watch`
   - Query Parameters:
     - `pageToken`: `={{ $json.startPageToken }}`
     - `supportsAllDrives`: `true`
     - `includeItemsFromAllDrives`: `true`
     - `driveId`: `={{ $('Set Variables').first().json.driveId }}`
   - Body (JSON):
     ```json
     {
       "id": "{{ $('Set Variables').first().json.channelId }}",
       "type": "web_hook",
       "address": "{{ $('Set Variables').first().json.webhookUrl }}",
       "token": "{{ $('Set Variables').first().json.channelToken }}",
       "expiration": {{ $now.plus({ "days": 7 }).toMillis() }}
     }
     ```
   - Authentication: Google Drive OAuth2
   - Connect "Get StartPageToken" to this node.

9. **Create Code Node**
   - Type: Code
   - Name: "Save Registration to WorkflowStaticData"
   - Purpose: Save registration info and startPageToken persistently.
   - Code:
     ```javascript
     const workflowStaticData = $getWorkflowStaticData('global');
     const registration = $input.item.json;
     const lastPageToken = $('Get StartPageToken').first().json.startPageToken;
     workflowStaticData.registration = registration;
     workflowStaticData.lastPageToken = lastPageToken;
     return { workflowStaticData };
     ```
   - Connect "Register Webhook" to this node.

10. **Create Webhook Node**
    - Type: Webhook
    - Name: "Webhook"
    - HTTP Method: POST
    - Path: Unique path (auto-generated or custom)
    - Purpose: Receive Google Drive push notifications.
    - This is the external endpoint Google calls.

11. **Create Code Node**
    - Type: Code
    - Name: "Get Registration From WorkflowStaticData"
    - Purpose: Retrieve registration data for validation.
    - Code:
      ```javascript
      const staticData = $getWorkflowStaticData('global');
      return {
        ...$input.item.json,
        workflowStaticData: staticData,
        driveId: staticData.registration.resourceUri.match(/driveId=([^&]+)&/)[1]
      };
      ```
    - Connect "Webhook" to this node.

12. **Create If Node**
    - Type: If
    - Name: "Is Valid Request?"
    - Conditions (all must be true):
      - `{{$json.headers['x-goog-channel-id']}}` equals `{{$json.workflowStaticData.registration.id}}`
      - `{{$json.headers['x-goog-channel-token']}}` equals `{{$json.workflowStaticData.registration.token}}`
      - `{{$json.headers['x-goog-resource-state']}}` equals `"change"`
    - Connect "Get Registration From WorkflowStaticData" to this node.
    - True branch: Connect to "Get Changes List"
    - False branch: Connect to "Invalid Request"

13. **Create No Operation Node**
    - Type: No Operation
    - Name: "Invalid Request"
    - Purpose: Placeholder to ignore invalid notifications.

14. **Create HTTP Request Node**
    - Type: HTTP Request
    - Name: "Get Changes List"
    - Method: GET
    - URL: `https://www.googleapis.com/drive/v3/changes`
    - Query Parameters:
      - `pageToken`: `={{ $json.workflowStaticData.lastPageToken || 10000 }}`
      - `supportsAllDrives`: `true`
      - `includeItemsFromAllDrives`: `true`
      - `driveId`: `={{ $json.driveId }}`
    - Authentication: Google Drive OAuth2
    - Connect "Is Valid Request?" (true branch) to this node.

15. **Create Code Node**
    - Type: Code
    - Name: "Save NextPageToken to WorkflowStaticData"
    - Purpose: Update lastPageToken after changes retrieval.
    - Code:
      ```javascript
      const workflowStaticData = $getWorkflowStaticData('global');
      const lastPageToken = $json.newStartPageToken || $json.nextPageToken;
      workflowStaticData.lastPageToken = lastPageToken;
      return { workflowStaticData };
      ```
    - Connect "Get Changes List" to this node.

16. **Create SplitOut Node**
    - Type: SplitOut
    - Name: "Change Events to Items"
    - Field to split: `changes`
    - Connect "Get Changes List" to this node.

17. **Create Filter Node**
    - Type: Filter
    - Name: "Filter Changed Events"
    - Conditions:
      - `removed` equals false
      - `type` equals `"file"`
      - `changeType` equals `"file"`
      - `file.mimeType` equals `"application/pdf"`
    - Connect "Change Events to Items" to this node.

18. **Create HTTP Request Node**
    - Type: HTTP Request
    - Name: "Get Files Details"
    - Method: GET
    - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.file.id }}`
    - Query Parameters:
      - `fields`: `id,kind,name,createdTime,modifiedTime,mimeType,trashed,driveId,parents`
      - `supportsAllDrives`: `true`
    - Authentication: Google Drive OAuth2
    - Connect "Filter Changed Events" to this node.

19. **Create No Operation Node**
    - Type: No Operation
    - Name: "Do Something With These Files!"
    - Purpose: Placeholder for custom file processing.
    - Connect "Get Files Details" to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow uses Google Drive Push Notifications for efficient file change monitoring. It requires Google Drive OAuth2 credentials and works with Shared Drives as well. | [Google Drive Push Notifications Guide](https://developers.google.com/workspace/drive/api/guides/push) |
| WorkflowStaticData persistence requires running the workflow in production mode, not editor mode. | [n8n WorkflowStaticData Docs](https://docs.n8n.io/code/cookbook/builtin/get-workflow-static-data/) |
| Google Drive Notification Channels expire after 7 days; renewal every 6 days is recommended to maintain active channels. | - |
| Push notifications can arrive multiple times in quick succession; ensure idempotency in downstream processing. | - |
| For support or community help, use the [n8n Discord](https://discord.com/invite/XPKeKXeB7d) or [n8n Forum](https://community.n8n.io/). | - |
| Banner image and branding are included in the workflow sticky notes for visual identification. | https://cdn.subworkflow.ai/n8n-templates/banner_595x311.png |

---

**Disclaimer:**  
The text and workflow described above originate exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled are legal and public.