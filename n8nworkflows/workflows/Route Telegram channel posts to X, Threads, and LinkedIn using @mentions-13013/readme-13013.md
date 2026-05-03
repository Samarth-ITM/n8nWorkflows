Route Telegram channel posts to X, Threads, and LinkedIn using @mentions

https://n8nworkflows.xyz/workflows/route-telegram-channel-posts-to-x--threads--and-linkedin-using--mentions-13013


# Route Telegram channel posts to X, Threads, and LinkedIn using @mentions

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically route Telegram **channel posts** to one or more social platforms—**X (Twitter)**, **Threads**, and **LinkedIn**—based on `@mentions` included in the Telegram message. It also sends Telegram notifications on success (per platform) or on routing/parsing errors.

**Primary use cases:**
- Single-message cross-posting from a Telegram channel to selected networks.
- Simple “operator control” via mentions: `@x`, `@threads`, `@linkedin`, or `@all`.
- Basic content cleanup before posting (removes routing mentions and hashtags).

### 1.1 Threads Token Setup (manual, one-time/periodic)
Manually executed branch to help obtain a long-lived Threads token (valid ~60 days), using the Threads API.

### 1.2 Input Reception (Telegram channel post)
Listens for `channel_post` updates from a Telegram bot that is admin in the channel.

### 1.3 Message Parsing & Routing Decision
Parses the incoming message text, converts platform mentions to a **bitwise flag**, cleans the message, and routes execution with a Switch.

### 1.4 Social Posting (X / Threads / LinkedIn)
Posts the cleaned message to the selected platform(s). Threads requires a two-step API flow: create then publish.

### 1.5 Notifications (success / error)
Sends a Telegram notification to a configured chat for: errors, X post success, Threads post success, LinkedIn post success.

---

## 2. Block-by-Block Analysis

### Block 1 — Threads Token Setup (Manual)
**Overview:** Provides a manual path to exchange a short-lived Threads token for a long-lived token. This is separated from the Telegram-triggered posting flow.

**Nodes involved:**
- When clicking 'Execute workflow'
- Threads Fields
- Get Threads Long-Lived Token
- Sticky Note
- Sticky Note5

#### Node: **When clicking 'Execute workflow'**
- **Type / role:** Manual Trigger; starts the token exchange branch.
- **Config:** No parameters.
- **Outputs:** To **Threads Fields**.
- **Edge cases:** None (manual start).

#### Node: **Threads Fields**
- **Type / role:** Set node; stores Threads credential material as plain fields.
- **Config choices:** Sets:
  - `access_token` = placeholder `YOUR_THREADS_ACCESS_TOKEN`
  - `client_secret` = placeholder `YOUR_THREADS_CLIENT_SECRET`
  - `account_id` = placeholder `YOUR_THREADS_ACCOUNT_ID`
  - `threads_id` = placeholder `YOUR_THREADS_USER_ID`
- **Outputs:** To **Get Threads Long-Lived Token**.
- **Edge cases / risks:**
  - Storing tokens in node parameters is insecure; prefer n8n credentials or environment variables.
  - Incorrect values cause downstream API errors.

#### Node: **Get Threads Long-Lived Token**
- **Type / role:** HTTP Request; calls Threads token exchange endpoint.
- **Config choices:**
  - URL built via expression:  
    `https://graph.threads.net/access_token?grant_type=th_exchange_token&client_secret={{ $json.client_secret }}&access_token={{ $json.access_token }}`
  - Method defaults to GET (since not specified).
- **Inputs:** Fields from **Threads Fields**.
- **Outputs:** Returns token exchange response (expected to include the long-lived access token).
- **Edge cases / failures:**
  - Threads API errors (invalid/expired token, wrong client secret).
  - Rate limits, network failures.
  - Response structure changes from API could break follow-up usage (not used elsewhere in this workflow, but important operationally).

#### Sticky Note: **Sticky Note**
Content (applies to the token setup area):
- “## Get Access Token … [Guide](https://www.youtube.com/watch?v=nC1-KZabm6U) … access token that lasts for 2 months…”

#### Sticky Note: **Sticky Note5**
Content:
- “Threads Token Setup … One-time setup: Run manually to refresh your Threads long-lived access token (valid for 60 days).”

---

### Block 2 — Telegram Input Reception
**Overview:** Listens for new posts in a Telegram channel and emits the post payload into the workflow.

**Nodes involved:**
- Telegram Trigger
- Sticky Note2

#### Node: **Telegram Trigger**
- **Type / role:** Telegram Trigger; entry point for the main automation.
- **Config choices:**
  - `updates`: `channel_post` (only channel posts, not messages).
- **Credentials:** Telegram API credential named **“Telegram Bot”** (must be a bot token).
- **Outputs:** To **Parse Telegram Message**.
- **Edge cases / failures:**
  - Bot not added as **admin** to the channel → no updates received.
  - Wrong webhook configuration / Telegram connectivity issues.
  - If the channel post doesn’t include `text` (e.g., media-only posts), downstream code may fail.

#### Sticky Note: **Sticky Note2**
Content:
- “Trigger & Parsing … Receives messages from Telegram channel and parses @mentions…”

---

### Block 3 — Message Parsing & Routing
**Overview:** Extracts `@mentions` to determine target platforms, removes routing tags and hashtags, then routes via a Switch using a numeric flag.

**Nodes involved:**
- Parse Telegram Message
- Switch

#### Node: **Parse Telegram Message**
- **Type / role:** Code node (JavaScript); parses and cleans Telegram message.
- **Key logic:**
  - Supported mentions map to bitwise flags:
    - `@x` → 1
    - `@threads` → 2
    - `@linkedin` → 4
    - `@all` → 7
  - Extract mentions with regex: `/@(\w+)/g`
  - For each mention:
    - If valid, ORs the flag and removes it from text.
    - If invalid, returns `flag = -1` and `parsedMessage = "Invalid channel: @<name>"`
  - Removes hashtags: `/#\w+/g`
  - Trims and normalizes whitespace.
- **Input read:** `const message = $input.first().json.channel_post.text`
- **Output:** Single item: `{ flag, parsedMessage }`
- **Connections:** Output to **Switch**.
- **Edge cases / failures:**
  - `channel_post.text` missing (media-only posts, captions stored elsewhere) → runtime error.
  - Mention parsing only matches `\w` (letters/numbers/underscore). Mentions with hyphens won’t match.
  - Multiple platform mentions are allowed; resulting flags can be 3 (X+Threads), 5 (X+LinkedIn), 6 (Threads+LinkedIn), etc. **However, the Switch only handles 3 explicitly** (X+Threads). Other combos are not routed.
  - Removes *all* hashtags; if hashtags are desired on X/LinkedIn, they will be stripped.

#### Node: **Switch**
- **Type / role:** Switch node; branches based on `flag`.
- **Config choices:** Numeric equality rules against `{{$json.flag}}` for:
  1. `-1` → error
  2. `1` → X only
  3. `2` → Threads only
  4. `4` → LinkedIn only
  5. `7` → all three
  6. `3` → X + Threads
- **Outputs / routing:**
  - `-1` → **Send Error Notification**
  - `1` → **Create Tweet**
  - `2` → **Threads Fields Copy**
  - `4` → **Create a post**
  - `7` → **Create Tweet**, **Create a post**, **Threads Fields Copy** (three parallel connections)
  - `3` → **Create Tweet**, **Threads Fields Copy**
- **Edge cases / failures:**
  - Flags not covered (e.g., `5` or `6`) will produce **no output** and silently do nothing.
  - If there are **no mentions**, flag remains `0` and is not handled.

---

### Block 4 — Social Media Posting
**Overview:** Sends the cleaned `parsedMessage` to X and LinkedIn using native nodes, and to Threads using HTTP requests (create + publish).

**Nodes involved:**
- Create Tweet
- Create a post
- Threads Fields Copy
- Create Thread
- Publish Thread
- Sticky Note3

#### Node: **Create Tweet**
- **Type / role:** Twitter node; creates a tweet on X.
- **Config choices:**
  - Text: `{{$json.parsedMessage}}` (from parsing output)
- **Inputs:** From **Switch**.
- **Outputs:** To **X Success Notification**.
- **Version:** Twitter node typeVersion 2.
- **Edge cases / failures:**
  - Credential/auth failures (OAuth2).
  - X API errors (rate limiting, duplicate content restrictions, text length limits).
  - If `parsedMessage` is empty after cleaning, X may reject.

#### Node: **Create a post** (LinkedIn)
- **Type / role:** LinkedIn node; creates a LinkedIn post.
- **Config choices:**
  - `person`: a specific LinkedIn person identifier (`zOhu_5WAJN`) configured in node.
  - Text: `{{$json.parsedMessage}}`
- **Inputs:** From **Switch**.
- **Outputs:** To **LinkedIn Success Notification**.
- **Edge cases / failures:**
  - LinkedIn OAuth issues, permissions not granted for posting.
  - Wrong `person` target (user mismatch to credential).
  - Empty or overly long content.

#### Node: **Threads Fields Copy**
- **Type / role:** Set node; duplicates Threads token/user fields for the posting branch.
- **Config choices:** Same placeholders as “Threads Fields”.
- **Inputs:** From **Switch** when Threads is selected.
- **Outputs:** To **Create Thread**.
- **Edge cases / failures:**
  - Same security and correctness concerns as “Threads Fields”.
  - Note: Fields set here are not actually used by the request URL (which contains `YOUR_THREADS_USER_ID` hard-coded); token is pulled from credentials (Bearer auth). This node is effectively redundant unless you extend the workflow.

#### Node: **Create Thread**
- **Type / role:** HTTP Request; creates a Threads post container.
- **Config choices:**
  - POST to: `https://graph.threads.net/v1.0/YOUR_THREADS_USER_ID/threads` (user id is hard-coded placeholder)
  - Authentication: `httpBearerAuth` (generic credential type)
  - Body parameters:
    - `text` = `{{ $('Parse Telegram Message').item.json.parsedMessage }}`
    - `media_type` = `TEXT`
- **Inputs:** From **Threads Fields Copy** (though body uses Parse node by reference).
- **Outputs:** To **Publish Thread** (expects response with an `id`).
- **Edge cases / failures:**
  - If URL still contains placeholder `YOUR_THREADS_USER_ID`, requests will fail.
  - Bearer token expired → 401/403.
  - Threads API may require additional parameters or permissions depending on account status.
  - If parsing returned empty text, API may reject.

#### Node: **Publish Thread**
- **Type / role:** HTTP Request; publishes the created Threads container.
- **Config choices:**
  - POST to: `https://graph.threads.net/v1.0/YOUR_THREADS_USER_ID/threads_publish` (hard-coded placeholder)
  - Authentication: same bearer auth approach (credential not shown here—must be configured or inherited per node settings)
  - Body:
    - `creation_id` = `{{$json.id}}` (from Create Thread response)
- **Inputs:** From **Create Thread**.
- **Outputs:** To **Threads Success Notification**.
- **Edge cases / failures:**
  - Missing/invalid `id` if Create Thread failed or response changed.
  - Publishing may fail due to moderation, rate limits, or token scope.

#### Sticky Note: **Sticky Note3**
Content:
- “Social Media Posting … Posts cleaned message content … based on parsed flags.”

---

### Block 5 — Notifications
**Overview:** Sends Telegram messages confirming success per platform, and an error message when parsing detects invalid channels.

**Nodes involved:**
- Send Error Notification
- X Success Notification
- LinkedIn Success Notification
- Threads Success Notification
- Sticky Note4

#### Node: **Send Error Notification**
- **Type / role:** Telegram node; sends error alerts.
- **Config choices:**
  - Text expression:  
    `Post on SM channels failed with error[{{ $('Parse Telegram Message').item.json.flag }}]:{{ $('Parse Telegram Message').item.json.parsedMessage }}`
  - `chatId`: placeholder `YOUR_NOTIFICATION_CHAT_ID`
  - `appendAttribution`: false
- **Credential:** Telegram API credential named **“Telgram Bot”** (note spelling differs from trigger’s “Telegram Bot”).
- **Inputs:** From Switch when `flag = -1`.
- **Edge cases / failures:**
  - Mis-typed credential name could indicate wrong credential selection.
  - If the notification chat ID is wrong, message won’t deliver.
  - Only parsing errors generate notifications; downstream posting errors are not caught/handled.

#### Node: **X Success Notification**
- **Type / role:** Telegram node; notifies success after tweet creation.
- **Text:** `✅ X | {{$json.id.substring(0,4) }} | {{ $json.text.substring(0,10) }}`
- **Inputs:** From **Create Tweet** (expects response has `id` and `text`).
- **Edge cases:** If X node response differs, substring calls can throw.

#### Node: **LinkedIn Success Notification**
- **Type / role:** Telegram node; notifies success after LinkedIn post.
- **Text:** `✅ LinkedIn | {{ $json.urn.substring(13, 17) }}`
- **Inputs:** From **Create a post** (expects response has `urn`).
- **Edge cases:** If `urn` shorter/unexpected, substring may error.

#### Node: **Threads Success Notification**
- **Type / role:** Telegram node; notifies success after Threads publish.
- **Text:** `✅ Threads | {{$json.id.substring(0,4) }} | {{ $('Switch').item.json.parsedMessage.substring(0,10) }}`
- **Inputs:** From **Publish Thread** (expects `id`).
- **Edge cases:**
  - `$('Switch').item.json.parsedMessage` assumes the “current item” alignment; in multi-branch situations, item linking can be fragile.
  - Substring calls can throw if values missing.

#### Sticky Note: **Sticky Note4**
Content:
- “Notifications … Sends confirmation messages back to Telegram…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual start for Threads token exchange | — | Threads Fields | ## Get Access Token… [Guide](https://www.youtube.com/watch?v=nC1-KZabm6U) … token lasts for 2 months… |
| Threads Fields | Set | Store Threads placeholders for token exchange | When clicking 'Execute workflow' | Get Threads Long-Lived Token | Threads Token Setup… Run manually to refresh long-lived access token (valid for 60 days). |
| Get Threads Long-Lived Token | HTTP Request | Exchange short-lived token for long-lived token | Threads Fields | — | ## Get Access Token… [Guide](https://www.youtube.com/watch?v=nC1-KZabm6U) … |
| Telegram Trigger | Telegram Trigger | Receive Telegram channel posts | — | Parse Telegram Message | Trigger & Parsing… parses @mentions… using binary flags. |
| Parse Telegram Message | Code | Parse mentions → flag, clean message | Telegram Trigger | Switch | Trigger & Parsing… parses @mentions… using binary flags. |
| Switch | Switch | Route to platforms based on flag | Parse Telegram Message | Send Error Notification / Create Tweet / Threads Fields Copy / Create a post | Social Media Posting… Posts cleaned message… based on parsed flags. |
| Create Tweet | Twitter | Post to X | Switch | X Success Notification | Social Media Posting… Posts cleaned message… based on parsed flags. |
| X Success Notification | Telegram | Notify X post success | Create Tweet | — | Notifications… confirmation messages back to Telegram… |
| Create a post | LinkedIn | Post to LinkedIn | Switch | LinkedIn Success Notification | Social Media Posting… Posts cleaned message… based on parsed flags. |
| LinkedIn Success Notification | Telegram | Notify LinkedIn post success | Create a post | — | Notifications… confirmation messages back to Telegram… |
| Threads Fields Copy | Set | Store Threads placeholders for posting branch | Switch | Create Thread | Social Media Posting… Posts cleaned message… based on parsed flags. |
| Create Thread | HTTP Request | Create Threads post container | Threads Fields Copy | Publish Thread | Social Media Posting… Posts cleaned message… based on parsed flags. |
| Publish Thread | HTTP Request | Publish Threads post container | Create Thread | Threads Success Notification | Social Media Posting… Posts cleaned message… based on parsed flags. |
| Threads Success Notification | Telegram | Notify Threads post success | Publish Thread | — | Notifications… confirmation messages back to Telegram… |
| Send Error Notification | Telegram | Notify parsing/routing error | Switch | — | Notifications… confirmation messages back to Telegram… |
| Sticky Note | Sticky Note | Documentation | — | — | ## Get Access Token… [Guide](https://www.youtube.com/watch?v=nC1-KZabm6U) … |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## How it works… Setup steps… Use @x/@threads/@linkedin/@all… |
| Sticky Note2 | Sticky Note | Documentation | — | — | Trigger & Parsing… |
| Sticky Note3 | Sticky Note | Documentation | — | — | Social Media Posting… |
| Sticky Note4 | Sticky Note | Documentation | — | — | Notifications… |
| Sticky Note5 | Sticky Note | Documentation | — | — | Threads Token Setup… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Send a Telegram message to three social media channels (X, Threads, LinkedIn)* (or your desired name).

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: **channel_post**
   - Credentials: create/select **Telegram Bot** credential (BotFather token).
   - Ensure the bot is **admin** in your Telegram channel.

3. **Add Code node: Parse Telegram Message**
   - Node: **Code**
   - Paste logic that:
     - Reads: `const message = $input.first().json.channel_post.text`
     - Converts `@x/@threads/@linkedin/@all` to bitwise flag (1/2/4/7)
     - Removes valid routing mentions and all hashtags
     - Outputs: `{ flag, parsedMessage }`
   - Connect: **Telegram Trigger → Parse Telegram Message**

4. **Add Switch node**
   - Node: **Switch**
   - Add number-equals rules on `{{$json.flag}}` for: `-1`, `1`, `2`, `4`, `7`, `3`
   - Connect: **Parse Telegram Message → Switch**

5. **Add X posting path**
   - Node: **Twitter** (operation: create tweet/post)
   - Text: `{{$json.parsedMessage}}`
   - Credentials: set up X/Twitter OAuth in n8n for the account.
   - Connect: **Switch (flag=1,7,3 outputs) → Create Tweet**

6. **Add LinkedIn posting path**
   - Node: **LinkedIn** (create post)
   - Target: select the posting identity (Person). Configure `person` to your profile in node settings.
   - Text: `{{$json.parsedMessage}}`
   - Credentials: LinkedIn OAuth2 credentials with posting permissions.
   - Connect: **Switch (flag=4,7 output) → Create a post**

7. **Add Threads posting path (create + publish)**
   - (Optional but matching the workflow) Add **Set** node “Threads Fields Copy” with placeholders:
     - `access_token`, `client_secret`, `account_id`, `threads_id`
   - Add **HTTP Request** node “Create Thread”:
     - Method: POST
     - URL: `https://graph.threads.net/v1.0/<YOUR_THREADS_USER_ID>/threads`
     - Auth: **HTTP Bearer Auth** credential (create one containing your Threads long-lived token)
     - Body params:
       - `text` = `{{ $('Parse Telegram Message').item.json.parsedMessage }}`
       - `media_type` = `TEXT`
   - Add **HTTP Request** node “Publish Thread”:
     - Method: POST
     - URL: `https://graph.threads.net/v1.0/<YOUR_THREADS_USER_ID>/threads_publish`
     - Auth: same Bearer credential
     - Body param: `creation_id` = `{{$json.id}}`
   - Connect:
     - **Switch (flag=2,7,3 outputs) → Threads Fields Copy → Create Thread → Publish Thread**

8. **Add Telegram notifications**
   - Create a Telegram chat/channel/user ID to receive notifications, set `YOUR_NOTIFICATION_CHAT_ID`.
   - Add **Telegram** node “Send Error Notification”:
     - Text: `Post on SM channels failed with error[{{ $('Parse Telegram Message').item.json.flag }}]:{{ $('Parse Telegram Message').item.json.parsedMessage }}`
     - chatId: your notification chat id
     - appendAttribution: false
     - Credentials: Telegram bot credential
     - Connect: **Switch (flag=-1 output) → Send Error Notification**
   - Add **Telegram** node “X Success Notification”:
     - Text: `✅ X | {{$json.id.substring(0,4) }} | {{ $json.text.substring(0,10) }}`
     - Connect: **Create Tweet → X Success Notification**
   - Add **Telegram** node “LinkedIn Success Notification”:
     - Text: `✅ LinkedIn | {{ $json.urn.substring(13, 17) }}`
     - Connect: **Create a post → LinkedIn Success Notification**
   - Add **Telegram** node “Threads Success Notification”:
     - Text: `✅ Threads | {{$json.id.substring(0,4) }} | {{ $('Switch').item.json.parsedMessage.substring(0,10) }}`
     - Connect: **Publish Thread → Threads Success Notification**

9. **Add the manual Threads token exchange branch (optional but included in the workflow)**
   - Add **Manual Trigger** node.
   - Add **Set** node “Threads Fields” setting `access_token` and `client_secret` placeholders.
   - Add **HTTP Request** “Get Threads Long-Lived Token”:
     - URL: `https://graph.threads.net/access_token?grant_type=th_exchange_token&client_secret={{ $json.client_secret }}&access_token={{ $json.access_token }}`
   - Connect: **Manual Trigger → Threads Fields → Get Threads Long-Lived Token**

10. **Replace placeholders**
   - Threads URLs: replace `<YOUR_THREADS_USER_ID>` (do not leave `YOUR_THREADS_USER_ID`).
   - Notification chat id: replace `YOUR_NOTIFICATION_CHAT_ID`.
   - Ensure the Threads bearer credential contains a valid **long-lived token**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Get Access Token… guide… access token lasts for 2 months so you can publish threads.” | https://www.youtube.com/watch?v=nC1-KZabm6U |
| “How it works… uses tags (@x/@threads/@linkedin/@all)… parses using bitwise flags… setup steps for Telegram bot, X, LinkedIn, Threads…” | Sticky note documentation embedded in workflow |
| Important operational limitation: Switch does not handle flags 5 (X+LinkedIn), 6 (Threads+LinkedIn), or 0 (no mentions). | Consider adding Switch rules or a default branch + notification |
| Posting error handling is not implemented for X/LinkedIn/Threads API failures (only parsing invalid channel triggers an error notification). | Add error workflows (Error Trigger) or IF nodes checking responses |