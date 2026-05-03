Get long-lived Facebook Page access tokens and subscribe Messenger webhook fields via Graph API

https://n8nworkflows.xyz/workflows/get-long-lived-facebook-page-access-tokens-and-subscribe-messenger-webhook-fields-via-graph-api-14027


# Get long-lived Facebook Page access tokens and subscribe Messenger webhook fields via Graph API

# 1. Workflow Overview

This workflow is a one-run Facebook Graph API utility for preparing Facebook Page automations, especially Messenger bots and Page reply systems. It performs two major setup tasks in sequence:

1. Exchanges a short-lived Facebook User Access Token for a long-lived User Access Token.
2. Uses that user context to retrieve Page Access Tokens for all Pages managed by the user, then subscribes each Page to selected webhook fields.

Typical use cases:
- Initial setup for Messenger chatbot projects
- Preparing Facebook Page comment/reply automations
- Periodic renewal of long-lived token and webhook subscriptions
- Bulk updating webhook subscriptions across multiple Pages connected to the same user

## 1.1 Input Reception and Configuration

The workflow starts manually and relies on a Set node where the operator must enter:
- Meta App ID
- Meta App Secret
- Short-lived User Access Token
- Desired webhook fields to subscribe

This block provides all runtime configuration used by the API calls that follow.

## 1.2 User Token Exchange and Identity Resolution

The workflow calls the Facebook Graph API OAuth endpoint to exchange the short-lived user token for a long-lived one. It then calls `/me` to resolve the app-scoped Facebook user ID associated with that token.

## 1.3 Page Access Token Retrieval

Using the app-scoped user ID and the long-lived user token, the workflow fetches all Pages linked to that user and retrieves their Page Access Tokens.

## 1.4 Per-Page Webhook Subscription Loop

The workflow splits the list of Pages into individual items, loops through them one by one, retrieves currently subscribed webhook fields for each Page, combines them with the fields requested by the operator, and posts the resulting field list back to Facebook.

## 1.5 Rate-Limit Protection

A 1-second wait is inserted between Page iterations to reduce the chance of Graph API throttling or burst-related errors.

---

# 2. Block-by-Block Analysis

## Block 1: Workflow Notes, Setup Guidance, and Entry Point

### Overview
This block contains the manual execution entry point and the documentation notes embedded directly in the canvas. It is the operator-facing starting area of the workflow and explains what must be configured before execution.

### Nodes Involved
- Main Overview
- Warning Edit
- Section 1
- Section 2
- Author Message
- Sticky Note
- When clicking 'Execute workflow'

### Node Details

#### Main Overview
- **Type and role:** Sticky Note; documentation-only node.
- **Configuration choices:** Contains a full explanation of the workflow purpose, setup steps, customization ideas, and licensing note.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None beyond standard sticky note support.
- **Edge cases or failure types:** None; informational only.
- **Sub-workflow reference:** None.

#### Warning Edit
- **Type and role:** Sticky Note; operator warning.
- **Configuration choices:** Highlights the four values that must be edited before execution: `app_id`, `app_secret`, `short_user_access_token`, `field_to_add`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Section 1
- **Type and role:** Sticky Note; section label.
- **Configuration choices:** Describes the token exchange and Page token retrieval logic.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Section 2
- **Type and role:** Sticky Note; section label.
- **Configuration choices:** Describes the per-Page subscription loop and wait strategy.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Author Message
- **Type and role:** Sticky Note; author attribution and related links.
- **Configuration choices:** Includes donation link, related workflow links, contact information, and creator page.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note
- **Type and role:** Sticky Note; related workflow references.
- **Configuration choices:** Lists additional n8n Facebook/Messenger-related workflows.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### When clicking 'Execute workflow'
- **Type and role:** `n8n-nodes-base.manualTrigger`; manual entry point.
- **Configuration choices:** No parameters; starts when the user clicks Execute Workflow.
- **Key expressions or variables used:** None.
- **Input and output connections:** No incoming connection; outgoing connection to `Needed Value`.
- **Version-specific requirements:** Type version 1; standard manual trigger.
- **Edge cases or failure types:** None at node level. Operational risk is only that the workflow will proceed with placeholder values if the Set node is not edited.
- **Sub-workflow reference:** None.

---

## Block 2: Runtime Configuration Input

### Overview
This block centralizes all runtime configuration values required for Graph API authentication and webhook subscription updates. It is the only node the operator must edit for functional execution.

### Nodes Involved
- Needed Value

### Node Details

#### Needed Value
- **Type and role:** `n8n-nodes-base.set`; initializes workflow variables.
- **Configuration choices:** Creates four string fields:
  - `app_id`
  - `app_secret`
  - `short_user_access_token`
  - `field_to_add`
- **Key expressions or variables used:** Static values entered manually by the user. Default placeholders are:
  - `[YOUR_APP_ID]`
  - `[YOUR_APP_SECRET]`
  - `[YOUR_SHORT_USER_ACCESS_TOKEN]`
  - `messages,messaging_postbacks,feed`
- **Input and output connections:** Input from `When clicking 'Execute workflow'`; output to `Get long-lived user access token`.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or failure types:**
  - Leaving placeholders unchanged will cause Facebook API authentication failures.
  - Invalid App ID / App Secret combination will cause token exchange failure.
  - Expired or malformed short-lived user token will cause OAuth errors.
  - `field_to_add` may include unsupported, duplicated, misspelled, or permission-sensitive fields.
- **Sub-workflow reference:** None.

---

## Block 3: User Token Exchange and User Identity Lookup

### Overview
This block converts the short-lived Facebook user token into a long-lived token and then resolves the current user’s app-scoped ID. That ID is required to fetch managed Pages from the `/accounts` endpoint.

### Nodes Involved
- Get long-lived user access token
- Get app scoped user id

### Node Details

#### Get long-lived user access token
- **Type and role:** `n8n-nodes-base.httpRequest`; calls the Facebook OAuth token exchange endpoint.
- **Configuration choices:**
  - Method defaults to GET
  - URL: `https://graph.facebook.com/v25.0/oauth/access_token`
  - Sends query parameters:
    - `grant_type=fb_exchange_token`
    - `client_id={{ $json.app_id }}`
    - `client_secret={{ $json.app_secret }}`
    - `fb_exchange_token={{ $json.short_user_access_token }}`
  - Full response enabled, so downstream nodes receive a wrapper containing status and `body`
- **Key expressions or variables used:**
  - `{{ $json.app_id }}`
  - `{{ $json.app_secret }}`
  - `{{ $json.short_user_access_token }}`
- **Input and output connections:** Input from `Needed Value`; output to `Get app scoped user id`.
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or failure types:**
  - Invalid app credentials
  - Invalid or expired short-lived token
  - Missing required permissions on the token
  - Facebook API version changes affecting endpoint behavior
  - Full response mode means the access token is under `body.access_token`; downstream expressions depend on that shape
- **Sub-workflow reference:** None.

#### Get app scoped user id
- **Type and role:** `n8n-nodes-base.httpRequest`; retrieves the current user profile using the long-lived token.
- **Configuration choices:**
  - URL expression: `https://graph.facebook.com/me?access_token={{ $json.body.access_token }}`
  - Uses the output of the previous node directly
- **Key expressions or variables used:**
  - `{{ $json.body.access_token }}`
- **Input and output connections:** Input from `Get long-lived user access token`; output to `Get long-lived page access token`.
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or failure types:**
  - If previous node failed or `body.access_token` is missing, expression evaluation fails or returns an invalid URL
  - Token may be valid but insufficient to read user context
  - Graph API may return a permissions-related error
- **Sub-workflow reference:** None.

---

## Block 4: Retrieve Managed Pages and Their Page Tokens

### Overview
This block uses the app-scoped user ID and the long-lived user token to list Facebook Pages available to the user. The resulting response includes Page-level access tokens, which are then split into individual Page items for processing.

### Nodes Involved
- Get long-lived page access token
- Split Out Pages

### Node Details

#### Get long-lived page access token
- **Type and role:** `n8n-nodes-base.httpRequest`; retrieves all managed Pages from the user’s `/accounts` edge.
- **Configuration choices:**
  - URL: `https://graph.facebook.com/v25.0/{{ $json.id }}/accounts`
  - Full response enabled
  - Query parameter:
    - `access_token={{ $('Get long-lived user access token').item.json.body.access_token }}`
- **Key expressions or variables used:**
  - `{{ $json.id }}`
  - `{{ $('Get long-lived user access token').item.json.body.access_token }}`
- **Input and output connections:** Input from `Get app scoped user id`; output to `Split Out Pages`.
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or failure types:**
  - If `/me` did not return `id`, the URL becomes invalid
  - Missing `pages_manage_metadata` or related Page permissions may prevent access
  - If the user manages no Pages, `body.data` may be empty
  - Downstream logic assumes the response body contains a `data` array
- **Sub-workflow reference:** None.

#### Split Out Pages
- **Type and role:** `n8n-nodes-base.splitOut`; converts the Page list array into one item per Page.
- **Configuration choices:**
  - `fieldToSplitOut = body.data`
- **Key expressions or variables used:** None; field path is configured statically.
- **Input and output connections:** Input from `Get long-lived page access token`; output to `Loop Over Items`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or failure types:**
  - If `body.data` is missing or not an array, the node may output nothing or fail depending on runtime data shape
  - Empty array means no Page iteration occurs
- **Sub-workflow reference:** None.

---

## Block 5: Per-Page Iteration and Current Subscription Lookup

### Overview
This block processes each Page one at a time. For each Page, it retrieves the current set of subscribed webhook fields via the `/subscribed_apps` edge.

### Nodes Involved
- Loop Over Items
- GET Current Fields

### Node Details

#### Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches`; sequential iteration controller.
- **Configuration choices:** Default batching behavior with no custom options shown. In this workflow it functions as a one-item loop controller.
- **Key expressions or variables used:** Downstream nodes reference the current loop item using:
  - `{{ $('Loop Over Items').item.json.id }}`
  - `{{ $('Loop Over Items').item.json.access_token }}`
- **Input and output connections:**
  - Input from `Split Out Pages`
  - Output to `GET Current Fields`
  - Receives control back from `Wait 1s (rate limit)` for continued iteration
- **Version-specific requirements:** Type version 3.
- **Edge cases or failure types:**
  - If no items are received, the block never executes
  - If item structure differs from expected Page object shape, downstream expressions break
  - If batching settings are later changed, loop behavior may differ
- **Sub-workflow reference:** None.

#### GET Current Fields
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches the current webhook subscriptions for the current Page.
- **Configuration choices:**
  - URL: `https://graph.facebook.com/v25.0/{{ $('Loop Over Items').item.json.id }}/subscribed_apps`
  - Query parameter:
    - `access_token={{ $('Loop Over Items').item.json.access_token }}`
- **Key expressions or variables used:**
  - `{{ $('Loop Over Items').item.json.id }}`
  - `{{ $('Loop Over Items').item.json.access_token }}`
- **Input and output connections:** Input from `Loop Over Items`; output to `Merge Fields`.
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or failure types:**
  - Page token may not permit reading subscriptions
  - Page may not yet be linked correctly to the app
  - Invalid or expired Page access token
  - API response shape may differ if no subscriptions exist or if the app is not installed for the Page
- **Sub-workflow reference:** None.

---

## Block 6: Merge Existing and Desired Webhook Fields, Then Update Subscription

### Overview
This block extracts existing subscribed fields from Facebook’s response and posts an updated field list back to the Graph API. It preserves current fields by converting them into a comma-separated string and appending the operator-provided fields.

### Nodes Involved
- Merge Fields
- POST Merged Fields

### Node Details

#### Merge Fields
- **Type and role:** `n8n-nodes-base.code`; transforms the GET response into a field string.
- **Configuration choices:**
  - JavaScript code reads `data` from the first input item
  - For each item in `data`, it reads `subscribed_fields`
  - It joins that array into a comma-separated string
  - Returns items shaped like `{ json: { result: "<comma-separated-fields>" } }`
- **Key expressions or variables used inside code:**
  - `$input.first().json.data`
  - `item.subscribed_fields || []`
- **Input and output connections:** Input from `GET Current Fields`; output to `POST Merged Fields`.
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential issues:**
  - If `json.data` is undefined or not an array, the code throws an error because `map` is called on a non-array
  - If Facebook returns multiple app subscription records, the node emits one output item per record; downstream behavior may then create multiple POST requests for the same Page
  - The node does not deduplicate fields
  - The node does not remove leading/trailing commas when no existing fields exist
- **Sub-workflow reference:** None.

#### POST Merged Fields
- **Type and role:** `n8n-nodes-base.httpRequest`; updates the Page’s webhook subscribed fields.
- **Configuration choices:**
  - Method: POST
  - URL: `https://graph.facebook.com/v25.0/{{ $('Loop Over Items').item.json.id }}/subscribed_apps`
  - Body content type: `application/x-www-form-urlencoded`
  - Body parameters:
    - `subscribed_fields={{ $json.result }},{{ $('Needed Value').item.json.field_to_add }}`
    - `access_token={{ $('Loop Over Items').item.json.access_token }}`
- **Key expressions or variables used:**
  - `{{ $json.result }}`
  - `{{ $('Needed Value').item.json.field_to_add }}`
  - `{{ $('Loop Over Items').item.json.access_token }}`
  - `{{ $('Loop Over Items').item.json.id }}`
- **Input and output connections:** Input from `Merge Fields`; output to `Wait 1s (rate limit)`.
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or failure types:**
  - Duplicate field names may be sent because the workflow concatenates rather than deduplicates
  - If `result` is empty, the outgoing string may begin with a comma
  - Invalid field names trigger Graph API validation errors
  - Missing app review approvals or Page permissions may block subscription updates
  - If the app is not properly configured for Messenger/webhook usage, POST may fail
- **Sub-workflow reference:** None.

---

## Block 7: Delay and Loop Continuation

### Overview
This block introduces a short pause before the workflow returns to the loop controller, helping reduce Graph API rate-limit issues when several Pages are processed.

### Nodes Involved
- Wait 1s (rate limit)

### Node Details

#### Wait 1s (rate limit)
- **Type and role:** `n8n-nodes-base.wait`; pauses execution briefly between Page updates.
- **Configuration choices:**
  - Wait amount: `1`
  - Used as a one-second delay
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `POST Merged Fields`; output back to `Loop Over Items`.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - In some n8n environments, Wait nodes require resumable execution support
  - If execution persistence is not configured correctly, paused executions may not resume as expected
  - One second may still be insufficient if many Pages are updated rapidly or Facebook applies stricter limits
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | General workflow explanation and setup guidance |  |  | ## Get Long-Lived Facebook Page Access Token & Subscribe Webhook Fields<br><br>This utility workflow automates two critical setup steps for any Facebook automation project: it exchanges a short-lived User Access Token for a **long-lived one**, retrieves **Page Access Tokens** for all connected Pages, then subscribes each Page to the webhook fields you need — all in one run.<br><br>If you're building a Messenger chatbot or automating Facebook Page comment replies, run this workflow once before activating your main automation.<br><br>### How it works<br>1. **Needed Value** — enter your App ID, App Secret, short-lived User Access Token, and the webhook fields to add.<br>2. **Token Exchange** — calls the Graph API to get a long-lived User Access Token, then resolves the app-scoped User ID.<br>3. **Page Tokens** — retrieves all Page Access Tokens linked to this user.<br>4. **Loop per Page** — for each Page, fetches its currently subscribed webhook fields, merges them with your new fields, and POSTs the combined list back to the Graph API.<br>5. **Rate Limit** — a 1-second Wait between pages prevents Graph API rate limit errors.<br><br>### Setup<br>* [ ] Get your **App ID** and **App Secret** from [Meta for Developers](https://developers.facebook.com/apps/).<br>* [ ] Generate a **short-lived User Access Token** using [Graph API Explorer](https://developers.facebook.com/tools/explorer/) with `pages_manage_metadata` permission.<br>* [ ] Fill in the **Needed Value** node with all four config fields.<br>* [ ] Run the workflow manually. Copy the **Page Access Token** from the output for use in your chatbot workflows.<br><br>### Customization tips<br>* Change `field_to_add` to any combination: `messages`, `messaging_postbacks`, `feed`, `message_reads`.<br>* Swap the Manual Trigger for a Schedule Trigger (every 50 days) to auto-refresh tokens.<br>* Add a Telegram node at the end to notify yourself when subscription is updated.<br><br>### LICENCE<br>This template is shared free of charge. Copyright belongs to Nguyen Thieu Toan (Jay Nguyen). Any copying or modification must credit the author. |
| Warning Edit | Sticky Note | Warning to edit required config fields |  |  | ## ⚠️ Edit this node!<br><br>Fill in all 4 fields before running:<br>- `app_id` — from Meta App Dashboard<br>- `app_secret` — from Meta App Dashboard<br>- `short_user_access_token` — from Graph API Explorer<br>- `field_to_add` — webhook fields to subscribe (e.g. `messages,feed`)<br><br>Get your token at [developers.facebook.com/tools/explorer](https://developers.facebook.com/tools/explorer) |
| Section 1 | Sticky Note | Visual label for token exchange section |  |  | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account. |
| Section 2 | Sticky Note | Visual label for per-page subscription section |  |  | ## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| Author Message | Sticky Note | Author attribution, donation, and related links |  |  | ## Author Message<br><br>Hi! I am **Nguyen Thieu Toan (Jay Nguyen)** — a Verified n8n Creator. Thank you for using this template!<br><br>This workflow is shared with you for free. If it brings value to your work, saves you time, or helps your automation projects, you can buy me a coffee here: **[My Donate Website](https://nguyenthieutoan.com/payment/)** *(PayPal, Momo, Bank Transfer)*<br><br>**Related workflows:**<br>- [Smart human takeover for Messenger chatbot](https://n8n.io/workflows/11920)<br>- [AI Facebook Messenger chatbot with Gemini](https://n8n.io/workflows/13080)<br>- [Smart message batching Messenger chatbot](https://n8n.io/workflows/9192)<br><br>* Website: [nguyenthieutoan.com](https://nguyenthieutoan.com)<br>* Email: me@nguyenthieutoan.com<br>* Company: GenStaff ([genstaff.net](https://genstaff.net))<br>* Socials: @nguyenthieutoan<br><br>*More templates:* **[n8n.io/creators/nguyenthieutoan](https://n8n.io/creators/nguyenthieutoan)** |
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow start |  | Needed Value | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account. |
| Needed Value | Set | Stores app credentials, token, and target fields | When clicking 'Execute workflow' | Get long-lived user access token | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account.<br>## ⚠️ Edit this node!<br><br>Fill in all 4 fields before running:<br>- `app_id` — from Meta App Dashboard<br>- `app_secret` — from Meta App Dashboard<br>- `short_user_access_token` — from Graph API Explorer<br>- `field_to_add` — webhook fields to subscribe (e.g. `messages,feed`)<br><br>Get your token at [developers.facebook.com/tools/explorer](https://developers.facebook.com/tools/explorer) |
| Get long-lived user access token | HTTP Request | Exchanges short-lived user token for long-lived user token | Needed Value | Get app scoped user id | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account. |
| Get app scoped user id | HTTP Request | Retrieves app-scoped Facebook user ID from long-lived token | Get long-lived user access token | Get long-lived page access token | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account. |
| Get long-lived page access token | HTTP Request | Lists managed Pages and their Page access tokens | Get app scoped user id | Split Out Pages | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account. |
| Split Out Pages | Split Out | Splits Page array into one item per Page | Get long-lived page access token | Loop Over Items | ## Section 1: Token Exchange<br>Exchanges the short-lived User Access Token → **long-lived User Access Token** (~60 days) → resolves app-scoped User ID → retrieves all **Page Access Tokens** linked to this user account.<br>## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| Loop Over Items | Split In Batches | Iterates Pages one by one | Split Out Pages, Wait 1s (rate limit) | GET Current Fields | ## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| GET Current Fields | HTTP Request | Reads current subscribed webhook fields for a Page | Loop Over Items | Merge Fields | ## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| Merge Fields | Code | Converts current subscribed fields array into CSV string | GET Current Fields | POST Merged Fields | ## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| POST Merged Fields | HTTP Request | Updates Page subscribed webhook fields | Merge Fields | Wait 1s (rate limit) | ## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| Wait 1s (rate limit) | Wait | Adds delay before next Page iteration | POST Merged Fields | Loop Over Items | ## Section 2: Per-Page Webhook Field Subscription<br>Splits pages → loops one by one → **GET** currently subscribed fields → **merges** with new fields from config → **POST** combined field list to Graph API. A 1-second Wait between iterations prevents rate limiting. |
| Sticky Note | Sticky Note | Related workflow links |  |  | ## **Related workflows**<br>- [Smart human takeover & auto pause AI-powered Facebook Messenger chatbot](https://n8n.io/workflows/11920)<br>- [Build a Facebook Messenger customer service AI chatbot with Google Gemini](https://n8n.io/workflows/13080)<br>- [Smart message batching AI-powered Facebook Messenger chatbot use Data Table](https://n8n.io/workflows/9192) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it: **Get long-lived Facebook Page access tokens and subscribe Messenger webhook fields via Graph API**.

2. **Add a Manual Trigger node**.
   - Node type: **Manual Trigger**
   - Leave default settings.
   - Rename it to: **When clicking 'Execute workflow'**

3. **Add a Set node** after the Manual Trigger.
   - Node type: **Set**
   - Rename it to: **Needed Value**
   - Add four string fields:
     1. `app_id` = `[YOUR_APP_ID]`
     2. `app_secret` = `[YOUR_APP_SECRET]`
     3. `short_user_access_token` = `[YOUR_SHORT_USER_ACCESS_TOKEN]`
     4. `field_to_add` = `messages,messaging_postbacks,feed`
   - Connect:
     - `When clicking 'Execute workflow'` → `Needed Value`

4. **Add an HTTP Request node** to exchange the token.
   - Node type: **HTTP Request**
   - Rename it to: **Get long-lived user access token**
   - Method: **GET**
   - URL: `https://graph.facebook.com/v25.0/oauth/access_token`
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `grant_type` = `fb_exchange_token`
     - `client_id` = `={{ $json.app_id }}`
     - `client_secret` = `={{ $json.app_secret }}`
     - `fb_exchange_token` = `={{ $json.short_user_access_token }}`
   - In **Options**, enable **Full Response**
   - Connect:
     - `Needed Value` → `Get long-lived user access token`

5. **Add another HTTP Request node** to resolve the user ID.
   - Node type: **HTTP Request**
   - Rename it to: **Get app scoped user id**
   - Method: **GET**
   - URL:
     `=https://graph.facebook.com/me?access_token={{ $json.body.access_token }}`
   - Connect:
     - `Get long-lived user access token` → `Get app scoped user id`

6. **Add a third HTTP Request node** to get Pages and Page access tokens.
   - Node type: **HTTP Request**
   - Rename it to: **Get long-lived page access token**
   - Method: **GET**
   - URL:
     `=https://graph.facebook.com/v25.0/{{ $json.id }}/accounts`
   - Enable **Send Query Parameters**
   - Add query parameter:
     - `access_token` = `={{ $('Get long-lived user access token').item.json.body.access_token }}`
   - In **Options**, enable **Full Response**
   - Connect:
     - `Get app scoped user id` → `Get long-lived page access token`

7. **Add a Split Out node** to separate Pages into items.
   - Node type: **Split Out**
   - Rename it to: **Split Out Pages**
   - Field to split out:
     - `body.data`
   - Connect:
     - `Get long-lived page access token` → `Split Out Pages`

8. **Add a Split In Batches node** for the loop.
   - Node type: **Split In Batches**
   - Rename it to: **Loop Over Items**
   - Keep default settings unless your n8n version requires explicitly setting batch size to `1`.
   - Connect:
     - `Split Out Pages` → `Loop Over Items`

9. **Add an HTTP Request node** to read current subscribed fields for each Page.
   - Node type: **HTTP Request**
   - Rename it to: **GET Current Fields**
   - Method: **GET**
   - URL:
     `=https://graph.facebook.com/v25.0/{{ $('Loop Over Items').item.json.id }}/subscribed_apps`
   - Enable **Send Query Parameters**
   - Add query parameter:
     - `access_token` = `={{ $('Loop Over Items').item.json.access_token }}`
   - Connect:
     - `Loop Over Items` → `GET Current Fields`

10. **Add a Code node** to extract current fields.
    - Node type: **Code**
    - Rename it to: **Merge Fields**
    - Language: **JavaScript**
    - Paste this logic:

      ```javascript
      // Build merged subscribed_fields list for this page
      const dataArray = $input.first().json.data;
      const results = dataArray.map(item => {
        const subscribed_fields = item.subscribed_fields || [];
        return subscribed_fields.join(",");
      });
      return results.map(r => ({ json: { result: r } }));
      ```

    - Connect:
      - `GET Current Fields` → `Merge Fields`

11. **Add an HTTP Request node** to update the subscription.
    - Node type: **HTTP Request**
    - Rename it to: **POST Merged Fields**
    - Method: **POST**
    - URL:
      `=https://graph.facebook.com/v25.0/{{ $('Loop Over Items').item.json.id }}/subscribed_apps`
    - Enable **Send Body**
    - Content Type: **Form URL Encoded**
    - Add body parameters:
      - `subscribed_fields` = `={{ $json.result }},{{ $('Needed Value').item.json.field_to_add }}`
      - `access_token` = `={{ $('Loop Over Items').item.json.access_token }}`
    - Connect:
      - `Merge Fields` → `POST Merged Fields`

12. **Add a Wait node** for rate limiting.
    - Node type: **Wait**
    - Rename it to: **Wait 1s (rate limit)**
    - Wait amount: **1 second**
    - Connect:
      - `POST Merged Fields` → `Wait 1s (rate limit)`

13. **Close the loop**.
    - Connect:
      - `Wait 1s (rate limit)` → `Loop Over Items`
    - This returns control to the loop controller so the next Page can be processed.

14. **Add optional sticky notes** for clarity.
    - Add notes for:
      - Main overview and setup instructions
      - Warning to edit config values
      - Section 1 label
      - Section 2 label
      - Author and related links
    - These are not required for execution.

15. **Prepare Facebook prerequisites before execution**.
    - In Meta for Developers, obtain:
      - **App ID**
      - **App Secret**
    - In **Graph API Explorer**, generate a short-lived User Access Token.
    - Ensure the token includes at least:
      - `pages_manage_metadata`
    - Depending on your exact use case and Graph API policies, additional permissions may be needed for specific webhook fields.

16. **Fill the Set node values**.
    - Replace all placeholders in `Needed Value`.
    - Example:
      - `app_id`: your Meta app ID
      - `app_secret`: your Meta app secret
      - `short_user_access_token`: the short-lived token from Graph API Explorer
      - `field_to_add`: comma-separated fields such as `messages,messaging_postbacks,feed`

17. **Run the workflow manually**.
    - Click **Execute workflow**.
    - Check:
      - `Get long-lived user access token` output for `body.access_token`
      - `Get long-lived page access token` output for `body.data`
      - `POST Merged Fields` output for Facebook success response

18. **Use the returned Page access tokens** in downstream Facebook automation workflows.
    - The Page tokens appear in the response from `Get long-lived page access token`.
    - Store them securely if they will be reused.

19. **Optional hardening improvements** if you rebuild it for production:
    - Add an **IF** node to validate placeholders are not still present
    - Add deduplication logic in the Code node before POSTing
    - Trim commas and whitespace from `field_to_add`
    - Add error handling branches for Graph API failures
    - Replace Manual Trigger with a **Schedule Trigger** every ~50 days if token refresh automation is desired

## Credential Configuration

This workflow does **not** use n8n credential objects in its current form. Authentication is passed explicitly as query/body parameters.

Required external credentials/values:
- Meta App ID
- Meta App Secret
- Facebook short-lived User Access Token

Recommended security improvement:
- Replace the Set node’s plain-text storage with more secure patterns such as environment variables or credential-backed HTTP requests where possible.

## Sub-workflow Setup

There are **no Sub-Workflow / Execute Workflow nodes** in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Meta App Dashboard for obtaining App ID and App Secret | https://developers.facebook.com/apps/ |
| Graph API Explorer for generating the short-lived User Access Token | https://developers.facebook.com/tools/explorer/ |
| Suggested permission mentioned in the workflow: `pages_manage_metadata` | Facebook Graph API token generation context |
| Suggested webhook field examples: `messages`, `messaging_postbacks`, `feed`, `message_reads` | Used in `field_to_add` |
| Suggested enhancement: replace Manual Trigger with a Schedule Trigger every 50 days | For long-lived token refresh automation |
| Suggested enhancement: add a Telegram node to notify when subscriptions are updated | Workflow customization idea |
| Author: Nguyen Thieu Toan (Jay Nguyen) | Workflow attribution |
| Donate link | https://nguyenthieutoan.com/payment/ |
| Website | https://nguyenthieutoan.com |
| Company | https://genstaff.net |
| Creator profile | https://n8n.io/creators/nguyenthieutoan |
| Related workflow: Smart human takeover for Messenger chatbot | https://n8n.io/workflows/11920 |
| Related workflow: AI Facebook Messenger chatbot with Gemini | https://n8n.io/workflows/13080 |
| Related workflow: Smart message batching Messenger chatbot | https://n8n.io/workflows/9192 |