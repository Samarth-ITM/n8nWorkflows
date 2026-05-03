Publish a social media post with Swonkie

https://n8nworkflows.xyz/workflows/publish-a-social-media-post-with-swonkie-14024


# Publish a social media post with Swonkie

# 1. Workflow Overview

This workflow publishes a social media post through the Swonkie Public API. It handles the complete process required by Swonkie: registering media, downloading and uploading the media file, confirming the upload, polling until Swonkie finishes processing the media, creating a post, validating it, and finally publishing it immediately or moving it to the scheduled stage.

Typical use cases:
- Publish a single image or video post to a Swonkie-connected social profile
- Automate social publishing from a public media URL
- Validate content before publishing to catch platform-specific issues early
- Use as a template for larger content pipelines

## 1.1 Trigger & Configuration
The workflow starts manually, then loads all runtime settings from a central configuration node. This includes API credentials, target profile, caption, stage, and media source.

## 1.2 Media Registration and Upload
Swonkie requires a multi-step upload process. The workflow first creates a media record in Swonkie, then downloads the media file into n8n as binary, uploads that binary to the returned Azure Blob Storage URL, and confirms the upload.

## 1.3 Media Processing Poll Loop
After upload confirmation, Swonkie processes the file asynchronously. The workflow waits 3 seconds, polls the media status, and loops until the media is either `SUCCESS` or `ERROR`.

## 1.4 Post Creation and Validation
Once media processing succeeds, the workflow creates a post in Swonkie using the uploaded media and the configured caption. It then validates the post through the API to ensure it satisfies platform rules.

## 1.5 Publishing / Stage Transition
If validation passes, the workflow changes the post stage to either `publishNow` or `schedule`. A final output node returns a normalized success payload.

## 1.6 Error Handling
Two explicit failure branches stop the workflow with an error:
- Media processing returned `ERROR`
- Post validation failed

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger & Configuration

### Overview
This block initializes the workflow and centralizes all editable values in one place. It is the only place a user must modify before running the workflow.

### Nodes Involved
- Start
- Configure

### Node Details

#### Start
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual workflow entry point
- **Configuration choices:** No parameters; execution starts manually from the editor or test run
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to `Configure`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None internally; only runs when manually triggered
- **Sub-workflow reference:** None

#### Configure
- **Type and technical role:** `n8n-nodes-base.set`; stores runtime configuration values as JSON fields
- **Configuration choices:**
  - `apiBase`: `https://api.swonkie.dev/v2`
  - `apiId`: placeholder for Swonkie App ID
  - `apiKey`: placeholder for Swonkie API Key
  - `profileId`: target social profile ID
  - `caption`: post text
  - `stage`: `publishNow` by default; alternative is `schedule`
  - `mediaUrl`: public file URL
  - `mediaName`: filename sent to Swonkie
- **Key expressions or variables used:** These fields are referenced throughout the workflow via expressions such as `$('Configure').item.json.apiBase`
- **Input and output connections:** Input from `Start`; output to `Create Media Entry`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged
  - Invalid `stage` causing a later API error
  - Non-public or inaccessible `mediaUrl`
  - Incorrect `profileId`
  - Credentials stored in plain text unless replaced with n8n credentials
- **Sub-workflow reference:** None

---

## Block 2 — Media Registration and Upload

### Overview
This block performs the upload sequence required by Swonkie. It creates a media entry, downloads the source file, uploads it to the provided blob URL, and then informs Swonkie that the upload is complete.

### Nodes Involved
- Create Media Entry
- Download & Attach Binary
- Upload File to Blob
- Confirm Upload

### Node Details

#### Create Media Entry
- **Type and technical role:** `n8n-nodes-base.httpRequest`; creates a Swonkie media resource and receives upload metadata
- **Configuration choices:**
  - Method: `POST`
  - URL: `{{ apiBase }}/media`
  - Sends JSON body with:
    - `destinationType: 'POST_MEDIA'`
    - `name: mediaName`
  - Custom headers:
    - `X-API-ID`
    - `X-API-KEY`
- **Key expressions or variables used:**
  - `$('Configure').item.json.apiBase`
  - `$('Configure').item.json.mediaName`
  - `$('Configure').item.json.apiId`
  - `$('Configure').item.json.apiKey`
- **Input and output connections:** Input from `Configure`; output to `Download & Attach Binary`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Authentication failure due to invalid headers
  - API base URL misconfigured
  - Invalid or empty media name
  - Swonkie API availability issues
- **Sub-workflow reference:** None

#### Download & Attach Binary
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches the source media file and stores it as binary data
- **Configuration choices:**
  - URL: `mediaUrl`
  - Response format set to file
  - Binary field name becomes `data`
- **Key expressions or variables used:** `$('Configure').item.json.mediaUrl`
- **Input and output connections:** Input from `Create Media Entry`; output to `Upload File to Blob`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - URL inaccessible, expired, blocked, or returning HTML instead of media
  - Large files causing timeout or memory pressure
  - Redirects or hotlink protections
  - Unsupported content source
- **Sub-workflow reference:** None

#### Upload File to Blob
- **Type and technical role:** `n8n-nodes-base.httpRequest`; uploads binary media data to the blob URL supplied by Swonkie
- **Configuration choices:**
  - Method: `PUT`
  - URL: `$('Create Media Entry').item.json.uploadUrl`
  - Content type: binary data
  - Input binary field: `data`
  - Header: `x-ms-blob-type: BlockBlob`
  - `neverError: true` enabled for response handling
- **Key expressions or variables used:** `$('Create Media Entry').item.json.uploadUrl`
- **Input and output connections:** Input from `Download & Attach Binary`; output to `Confirm Upload`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Missing binary field if previous node did not produce file output
  - Upload URL expired or malformed
  - Azure Blob Storage rejects the request
  - Silent HTTP failure risk because `neverError` is enabled; downstream confirmation may fail later instead
- **Sub-workflow reference:** None

#### Confirm Upload
- **Type and technical role:** `n8n-nodes-base.httpRequest`; tells Swonkie the file upload is complete so processing can start
- **Configuration choices:**
  - Method: `PATCH`
  - URL: `{{ apiBase }}/media/{{ mediaId }}`
  - Uses the same API headers as other Swonkie endpoints
- **Key expressions or variables used:**
  - `$('Configure').item.json.apiBase`
  - `$('Create Media Entry').item.json.id`
  - `$('Configure').item.json.apiId`
  - `$('Configure').item.json.apiKey`
- **Input and output connections:** Input from `Upload File to Blob`; output to `Wait for Processing`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Blob upload actually failed but confirmation still attempted
  - Invalid media ID reference
  - Authentication issues
- **Sub-workflow reference:** None

---

## Block 3 — Media Processing Poll Loop

### Overview
This block waits and polls until Swonkie finishes asynchronous media processing. It branches on success, error, or ongoing processing.

### Nodes Involved
- Wait for Processing
- Check Media Status
- Media Ready?

### Node Details

#### Wait for Processing
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution for a short polling interval
- **Configuration choices:**
  - Wait amount: `3` seconds
  - Has a fixed webhook ID in node metadata
- **Key expressions or variables used:** None
- **Input and output connections:** Input from `Confirm Upload` and `Media Ready?` fallback output; output to `Check Media Status`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - Infinite loop possible if media remains stuck in a non-terminal state
  - Long executions can consume runtime resources
- **Sub-workflow reference:** None

#### Check Media Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; retrieves current status of the Swonkie media item
- **Configuration choices:**
  - Method defaults to `GET`
  - URL: `{{ apiBase }}/media/{{ mediaId }}`
  - Includes `X-API-ID` and `X-API-KEY`
- **Key expressions or variables used:**
  - `$('Configure').item.json.apiBase`
  - `$('Create Media Entry').item.json.id`
  - `$('Configure').item.json.apiId`
  - `$('Configure').item.json.apiKey`
- **Input and output connections:** Input from `Wait for Processing`; output to `Media Ready?`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - API transient errors during polling
  - Authentication failures
  - Unexpected response shape without `status`
- **Sub-workflow reference:** None

#### Media Ready?
- **Type and technical role:** `n8n-nodes-base.switch`; routes flow by media processing status
- **Configuration choices:**
  - Output `SUCCESS` when `$json.status === 'SUCCESS'`
  - Output `ERROR` when `$json.status === 'ERROR'`
  - Fallback output used for all other statuses, effectively treating them as “keep waiting”
- **Key expressions or variables used:** `={{ $json.status }}`
- **Input and output connections:**
  - Input from `Check Media Status`
  - Output 0 `SUCCESS` → `Create Post`
  - Output 1 `ERROR` → `Media Processing Failed`
  - Fallback output → `Wait for Processing`
- **Version-specific requirements:** Switch version `3.2`
- **Edge cases or potential failure types:**
  - Any unexpected status value loops forever
  - Missing `status` field also falls to fallback behavior
  - Case-sensitive matching means `success` or `error` would not match
- **Sub-workflow reference:** None

---

## Block 4 — Post Creation and Validation

### Overview
After successful media processing, the workflow creates a media post and validates it using Swonkie’s validation endpoint. Validation prevents invalid posts from being published.

### Nodes Involved
- Create Post
- Validate Post
- Post Valid?

### Node Details

#### Create Post
- **Type and technical role:** `n8n-nodes-base.httpRequest`; creates a new post resource in Swonkie
- **Configuration choices:**
  - Method: `POST`
  - URL: `{{ apiBase }}/posts`
  - JSON payload includes:
    - `type: 'MEDIA'`
    - `profileIds: [profileId]`
    - `captions: [{ plainText: caption, net: null }]`
    - `medias: [{ mediaFiles: [{ mediaLibId: mediaId }], net: null }]`
    - Empty `links` and `labelIds`
  - Uses API header authentication
- **Key expressions or variables used:**
  - `$('Configure').item.json.apiBase`
  - `$('Configure').item.json.profileId`
  - `$('Configure').item.json.caption`
  - `$('Create Media Entry').item.json.id`
  - `$('Configure').item.json.apiId`
  - `$('Configure').item.json.apiKey`
- **Input and output connections:** Input from `Media Ready?` success output; output to `Validate Post`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Invalid profile ID
  - Caption too long for later validation
  - Media reference not accepted
  - API auth or permission issues
- **Sub-workflow reference:** None

#### Validate Post
- **Type and technical role:** `n8n-nodes-base.httpRequest`; checks whether the created post can be published
- **Configuration choices:**
  - URL: `{{ apiBase }}/posts/{{ postId }}/validate`
  - `neverError: true` so HTTP 400 responses can still be inspected instead of hard-failing the node
  - Uses API headers
- **Key expressions or variables used:**
  - `$('Configure').item.json.apiBase`
  - `$json.id` from `Create Post`
  - `$('Configure').item.json.apiId`
  - `$('Configure').item.json.apiKey`
- **Input and output connections:** Input from `Create Post`; output to `Post Valid?`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Validation endpoint returns nonstandard payload
  - `neverError` means downstream logic must correctly interpret invalid responses
  - Network/auth issues still possible
- **Sub-workflow reference:** None

#### Post Valid?
- **Type and technical role:** `n8n-nodes-base.if`; tests whether validation result is explicitly true
- **Configuration choices:**
  - Boolean condition checks `$json.valid` is true
  - True branch continues to stage change
  - False branch stops with detailed validation error
- **Key expressions or variables used:** `={{ $json.valid }}`
- **Input and output connections:**
  - Input from `Validate Post`
  - True output → `Change Stage`
  - False output → `Validation Failed`
- **Version-specific requirements:** IF node version `2.2`
- **Edge cases or potential failure types:**
  - Missing `valid` field evaluates as false
  - Validation response schema changes could misroute data
- **Sub-workflow reference:** None

---

## Block 5 — Publishing / Stage Transition

### Overview
This block moves the post into its final execution stage and returns a simplified success payload for downstream use.

### Nodes Involved
- Change Stage
- Post Published

### Node Details

#### Change Stage
- **Type and technical role:** `n8n-nodes-base.httpRequest`; changes the Swonkie post stage to publish or schedule
- **Configuration choices:**
  - Method: `PATCH`
  - URL pattern: `{{ apiBase }}/posts/{{ postId }}/stage/{{ stage }}`
  - `stage` comes from configuration and is expected to be `publishNow` or `schedule`
  - Uses API headers
- **Key expressions or variables used:**
  - `$('Configure').item.json.apiBase`
  - `$('Create Post').item.json.id`
  - `$('Configure').item.json.stage`
  - `$('Configure').item.json.apiId`
  - `$('Configure').item.json.apiKey`
- **Input and output connections:** Input from `Post Valid?` true branch; output to `Post Published`
- **Version-specific requirements:** HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - `schedule` may fail if `publishAt` was not set earlier
  - Unsupported or mistyped stage
  - Swonkie may accept the stage request but actual platform publication can still fail later
- **Sub-workflow reference:** None

#### Post Published
- **Type and technical role:** `n8n-nodes-base.set`; formats final success output
- **Configuration choices:** Sets:
  - `success = true`
  - `message = "Post successfully handed off to Swonkie."`
  - `postId = Create Post.id`
  - `stage = Configure.stage`
- **Key expressions or variables used:**
  - `$('Create Post').item.json.id`
  - `$('Configure').item.json.stage`
- **Input and output connections:** Input from `Change Stage`; terminal output
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If earlier nodes failed, this node is never reached
  - Success means handoff to Swonkie, not necessarily final network delivery confirmation
- **Sub-workflow reference:** None

---

## Block 6 — Error Handling

### Overview
This block explicitly fails the workflow when media processing or validation cannot complete successfully. These failures are visible in n8n execution history and can trigger a configured error workflow.

### Nodes Involved
- Media Processing Failed
- Validation Failed

### Node Details

#### Media Processing Failed
- **Type and technical role:** `n8n-nodes-base.stopAndError`; stops execution and marks it as failed
- **Configuration choices:**
  - Error message includes the media ID from `Create Media Entry`
- **Key expressions or variables used:** `$('Create Media Entry').item.json.id`
- **Input and output connections:** Input from `Media Ready?` error output; no output
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Message is generic and does not include full API diagnostics
  - Used only when Swonkie returns `ERROR` status
- **Sub-workflow reference:** None

#### Validation Failed
- **Type and technical role:** `n8n-nodes-base.stopAndError`; stops execution with details from validation response
- **Configuration choices:**
  - Error message includes post ID and stringified validation response
- **Key expressions or variables used:**
  - `$('Create Post').item.json.id`
  - `JSON.stringify($json)`
- **Input and output connections:** Input from `Post Valid?` false branch; no output
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Large validation payloads may produce long log messages
  - If validation response is malformed, error detail quality may be reduced
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start | Manual Trigger | Manual entry point |  | Configure | ## Trigger & Configure |
| Configure | Set | Central runtime configuration | Start | Create Media Entry | ## ⚙️ Setup - Edit This Node First |
| Configure | Set | Central runtime configuration | Start | Create Media Entry | ⚠️ Security note: Credentials are stored as plain text in the workflow JSON and visible in execution logs. For production use, create a **Generic Credential → Header Auth** in n8n's Credentials panel with `X-API-ID` and `X-API-KEY` as headers, then reference it in each HTTP Request node instead. |
| Configure | Set | Central runtime configuration | Start | Create Media Entry | ## Trigger & Configure |
| Create Media Entry | HTTP Request | Register media and get upload URL | Configure | Download & Attach Binary | ## 📤 Media Upload |
| Create Media Entry | HTTP Request | Register media and get upload URL | Configure | Download & Attach Binary | ## Media Upload |
| Download & Attach Binary | HTTP Request | Download source media as binary | Create Media Entry | Upload File to Blob | ## 📤 Media Upload |
| Download & Attach Binary | HTTP Request | Download source media as binary | Create Media Entry | Upload File to Blob | ## Media Upload |
| Upload File to Blob | HTTP Request | Upload binary to Azure Blob Storage | Download & Attach Binary | Confirm Upload | ## 📤 Media Upload |
| Upload File to Blob | HTTP Request | Upload binary to Azure Blob Storage | Download & Attach Binary | Confirm Upload | ## Media Upload |
| Confirm Upload | HTTP Request | Confirm upload completion to Swonkie | Upload File to Blob | Wait for Processing | ## 📤 Media Upload |
| Confirm Upload | HTTP Request | Confirm upload completion to Swonkie | Upload File to Blob | Wait for Processing | ## Media Upload |
| Wait for Processing | Wait | Delay before status polling | Confirm Upload; Media Ready? | Check Media Status | ## 📤 Media Upload |
| Wait for Processing | Wait | Delay before status polling | Confirm Upload; Media Ready? | Check Media Status | ## Media Upload |
| Check Media Status | HTTP Request | Poll media processing state | Wait for Processing | Media Ready? | ## 📤 Media Upload |
| Check Media Status | HTTP Request | Poll media processing state | Wait for Processing | Media Ready? | ## Media Upload |
| Media Ready? | Switch | Route by media status | Check Media Status | Create Post; Media Processing Failed; Wait for Processing | ## 📤 Media Upload |
| Media Ready? | Switch | Route by media status | Check Media Status | Create Post; Media Processing Failed; Wait for Processing | ## Media Upload |
| Create Post | HTTP Request | Create Swonkie post | Media Ready? | Validate Post | ## 📝 Post Publishing |
| Create Post | HTTP Request | Create Swonkie post | Media Ready? | Validate Post | ## Create Post |
| Validate Post | HTTP Request | Validate created post | Create Post | Post Valid? | ## 📝 Post Publishing |
| Validate Post | HTTP Request | Validate created post | Create Post | Post Valid? | ## Create Post |
| Post Valid? | If | Branch on validation result | Validate Post | Change Stage; Validation Failed | ## 📝 Post Publishing |
| Post Valid? | If | Branch on validation result | Validate Post | Change Stage; Validation Failed | ## Create Post |
| Change Stage | HTTP Request | Move post to final stage | Post Valid? | Post Published | ## Publish Post |
| Post Published | Set | Produce final success payload | Change Stage |  | ## Publish Post |
| Media Processing Failed | Stop and Error | Fail workflow on media processing error | Media Ready? |  | ## ❌ Error Handling |
| Validation Failed | Stop and Error | Fail workflow on validation error | Post Valid? |  | ## ❌ Error Handling |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `Publish a social media post with Swonkie`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name: `Start`

3. **Add a Set node**
   - Node type: `Set`
   - Name: `Configure`
   - Add these fields:
     - `apiBase` as string: `https://api.swonkie.dev/v2`
     - `apiId` as string: your Swonkie App ID
     - `apiKey` as string: your Swonkie API Key
     - `profileId` as string: target profile ID
     - `caption` as string: post text
     - `stage` as string: `publishNow`
     - `mediaUrl` as string: public URL to an image or video
     - `mediaName` as string: e.g. `post-image.png`
   - Optional node note: `Fill in your credentials and post settings here.`

4. **Connect `Start` → `Configure`**

5. **Add an HTTP Request node**
   - Name: `Create Media Entry`
   - Method: `POST`
   - URL: `={{ $('Configure').item.json.apiBase }}/media`
   - Enable **Send Headers**
   - Add headers:
     - `X-API-ID` = `={{ $('Configure').item.json.apiId }}`
     - `X-API-KEY` = `={{ $('Configure').item.json.apiKey }}`
   - Enable **Send Body**
   - Body type: JSON
   - JSON body:
     - `destinationType` = `POST_MEDIA`
     - `name` = `={{ $('Configure').item.json.mediaName }}`
   - In expression form, the original workflow uses `JSON.stringify(...)`; in the UI you can also build equivalent JSON fields directly if supported.

6. **Connect `Configure` → `Create Media Entry`**

7. **Add another HTTP Request node**
   - Name: `Download & Attach Binary`
   - URL: `={{ $('Configure').item.json.mediaUrl }}`
   - Response format: **File**
   - This makes the downloaded file available as binary, typically under the `data` property

8. **Connect `Create Media Entry` → `Download & Attach Binary`**

9. **Add another HTTP Request node**
   - Name: `Upload File to Blob`
   - Method: `PUT`
   - URL: `={{ $('Create Media Entry').item.json.uploadUrl }}`
   - Enable **Send Body**
   - Body content type: `Binary Data`
   - Binary property: `data`
   - Enable **Send Headers**
   - Add header:
     - `x-ms-blob-type` = `BlockBlob`
   - In options, set response handling so the node does not fail immediately on non-2xx responses (`neverError: true` equivalent)

10. **Connect `Download & Attach Binary` → `Upload File to Blob`**

11. **Add another HTTP Request node**
   - Name: `Confirm Upload`
   - Method: `PATCH`
   - URL: `={{ $('Configure').item.json.apiBase + '/media/' + $('Create Media Entry').item.json.id }}`
   - Enable **Send Headers**
   - Add headers:
     - `X-API-ID` = `={{ $('Configure').item.json.apiId }}`
     - `X-API-KEY` = `={{ $('Configure').item.json.apiKey }}`

12. **Connect `Upload File to Blob` → `Confirm Upload`**

13. **Add a Wait node**
   - Name: `Wait for Processing`
   - Wait duration: `3 seconds`

14. **Connect `Confirm Upload` → `Wait for Processing`**

15. **Add another HTTP Request node**
   - Name: `Check Media Status`
   - Method: `GET`
   - URL: `={{ $('Configure').item.json.apiBase + '/media/' + $('Create Media Entry').item.json.id }}`
   - Enable **Send Headers**
   - Add headers:
     - `X-API-ID` = `={{ $('Configure').item.json.apiId }}`
     - `X-API-KEY` = `={{ $('Configure').item.json.apiKey }}`

16. **Connect `Wait for Processing` → `Check Media Status`**

17. **Add a Switch node**
   - Name: `Media Ready?`
   - Evaluate field: `={{ $json.status }}`
   - Rule 1:
     - equals `SUCCESS`
     - rename output to `SUCCESS`
   - Rule 2:
     - equals `ERROR`
     - rename output to `ERROR`
   - Enable a fallback output for all other values

18. **Connect `Check Media Status` → `Media Ready?`**

19. **Create the poll loop**
   - Connect `Media Ready?` fallback output → `Wait for Processing`
   - This loops until media status becomes `SUCCESS` or `ERROR`

20. **Add a Stop and Error node**
   - Name: `Media Processing Failed`
   - Error message:
     - `=Media processing failed for mediaId {{ $('Create Media Entry').item.json.id }}. The file may be corrupt or in an unsupported format.`

21. **Connect `Media Ready?` error output → `Media Processing Failed`**

22. **Add an HTTP Request node**
   - Name: `Create Post`
   - Method: `POST`
   - URL: `={{ $('Configure').item.json.apiBase }}/posts`
   - Enable **Send Headers**
   - Add:
     - `X-API-ID` = `={{ $('Configure').item.json.apiId }}`
     - `X-API-KEY` = `={{ $('Configure').item.json.apiKey }}`
   - Enable **Send Body**
   - Body type: JSON
   - Build a payload equivalent to:
     - `type`: `MEDIA`
     - `profileIds`: array containing `profileId`
     - `captions`: array with one object:
       - `plainText`: `caption`
       - `net`: `null`
     - `medias`: array with one object:
       - `mediaFiles`: array with one object:
         - `mediaLibId`: media ID from `Create Media Entry`
       - `net`: `null`
     - `links`: empty array
     - `labelIds`: empty array

23. **Connect `Media Ready?` success output → `Create Post`**

24. **Add another HTTP Request node**
   - Name: `Validate Post`
   - Method: `GET` if Swonkie endpoint expects it as configured by n8n default behavior on that URL; keep the endpoint exactly as used here:
     - `={{ $('Configure').item.json.apiBase + '/posts/' + $json.id + '/validate' }}`
   - Enable **Send Headers**
   - Add:
     - `X-API-ID` = `={{ $('Configure').item.json.apiId }}`
     - `X-API-KEY` = `={{ $('Configure').item.json.apiKey }}`
   - Set response handling to not automatically fail on validation errors (`neverError: true` equivalent)

25. **Connect `Create Post` → `Validate Post`**

26. **Add an If node**
   - Name: `Post Valid?`
   - Condition:
     - Value 1: `={{ $json.valid }}`
     - Operation: `is true`

27. **Connect `Validate Post` → `Post Valid?`**

28. **Add a Stop and Error node**
   - Name: `Validation Failed`
   - Error message:
     - `=Post validation failed for postId {{ $('Create Post').item.json.id }}. Details: {{ JSON.stringify($json) }}`

29. **Connect `Post Valid?` false output → `Validation Failed`**

30. **Add another HTTP Request node**
   - Name: `Change Stage`
   - Method: `PATCH`
   - URL:
     - `={{ $('Configure').item.json.apiBase + '/posts/' + $('Create Post').item.json.id + '/stage/' + $('Configure').item.json.stage }}`
   - Enable **Send Headers**
   - Add:
     - `X-API-ID` = `={{ $('Configure').item.json.apiId }}`
     - `X-API-KEY` = `={{ $('Configure').item.json.apiKey }}`

31. **Connect `Post Valid?` true output → `Change Stage`**

32. **Add a Set node**
   - Name: `Post Published`
   - Add fields:
     - `success` as boolean = `true`
     - `message` as string = `Post successfully handed off to Swonkie.`
     - `postId` as string = `={{ $('Create Post').item.json.id }}`
     - `stage` as string = `={{ $('Configure').item.json.stage }}`

33. **Connect `Change Stage` → `Post Published`**

34. **Configure credentials securely for production**
   - Recommended approach:
     - Create a generic HTTP header credential in n8n
     - Add headers:
       - `X-API-ID`
       - `X-API-KEY`
     - Replace inline header expressions in all Swonkie HTTP nodes with that credential reference
   - Nodes affected:
     - `Create Media Entry`
     - `Confirm Upload`
     - `Check Media Status`
     - `Create Post`
     - `Validate Post`
     - `Change Stage`

35. **Add optional sticky notes**
   - Overview note describing the whole workflow
   - Setup note warning users to edit the `Configure` node first
   - Media Upload note describing the 3-step plus polling process
   - Post Publishing note describing create/validate/publish
   - Error Handling note describing failure behavior

36. **Run a test execution**
   - Confirm the `mediaUrl` is publicly accessible
   - Confirm the `profileId` exists and is connected in Swonkie
   - Confirm `stage` is valid:
     - `publishNow` for immediate publishing
     - `schedule` only if the post has already been assigned a `publishAt` date through the API

### Input expectations
- A public, downloadable file URL
- Valid Swonkie API credentials
- A valid Swonkie social profile ID

### Output expectations
On success, the final node returns:
- `success: true`
- `message`
- `postId`
- `stage`

On failure, the workflow stops with an execution error and an explanatory message.

### Important implementation constraints
- The media polling loop has no maximum retry count; consider adding one in production
- `neverError: true` is used in selected nodes to inspect non-2xx responses rather than failing immediately
- The workflow assumes a single-item execution path
- Scheduling is incomplete unless you separately set `publishAt` before moving the post to `schedule`

### Sub-workflow setup
- This workflow does **not** invoke any sub-workflow.
- There are **no** `Execute Workflow` nodes or multiple workflow entry points beyond the single manual trigger.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Swonkie API credentials can be obtained from the workspace public API settings page. | https://app.swonkie.com/settings/workspace/public-api |
| The target social Profile ID should be retrieved from the Swonkie `GET /profiles` endpoint. | Swonkie Public API usage |
| For production use, avoid plain-text credentials in Set nodes and execution logs; prefer n8n credentials with header auth. | n8n credential management |
| `publishNow` is the default stage. `schedule` should only be used if a `publishAt` value was set previously through the API. | Swonkie post lifecycle |
| A successful workflow run means the post was handed off to Swonkie. Final delivery to the social network may still depend on Swonkie processing and profile connectivity. | Operational behavior |