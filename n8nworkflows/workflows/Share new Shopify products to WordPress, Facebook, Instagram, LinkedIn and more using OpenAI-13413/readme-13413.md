Share new Shopify products to WordPress, Facebook, Instagram, LinkedIn and more using OpenAI

https://n8nworkflows.xyz/workflows/share-new-shopify-products-to-wordpress--facebook--instagram--linkedin-and-more-using-openai-13413


# Share new Shopify products to WordPress, Facebook, Instagram, LinkedIn and more using OpenAI

## 1. Workflow Overview

**Workflow name:** Automatically Share Shopify New Products Across All Social Media Platforms  
**Template title (given):** Share new Shopify products to WordPress, Facebook, Instagram, LinkedIn and more using OpenAI

**Purpose:**  
Automatically reacts to **new Shopify products**, derives a usable **image URL**, builds a **product URL**, uses **OpenAI** to shorten/transform the product description, then publishes the content across multiple channels (WordPress, Facebook, Instagram, LinkedIn, Telegram, Discord). It also includes **notification** branches for WhatsApp/email/Discord/Teams.

**Target use cases:**
- E-commerce stores that want immediate multi-channel promotion when a product is created in Shopify.
- Teams that need an automated social posting pipeline with AI-generated copy.
- Content syndication from Shopify → WordPress + social networks.

### 1.1 Entry + Image URL Detection
Triggered by Shopify; attempts to detect and/or split image link(s) for further processing.

### 1.2 Looping / Item Handling + Product URL Creation
Optionally loops items (for multiple images/variants) and creates a product URL.

### 1.3 AI Processing (OpenAI)
Converts Shopify HTML description into a short, social-friendly version (and uses Shopify product as an AI tool input).

### 1.4 Content Distribution (Image fetch → multi-platform posts)
Downloads the image and posts/creates content across WordPress and social platforms.

### 1.5 WordPress Media Handling (featured image pipeline)
Downloads + edits image, uploads it to WordPress, attaches metadata, sets featured image.

### 1.6 Notifications / Fallback Paths
Sends notifications to Telegram, Gmail, Discord, WhatsApp (Rapiwa), Teams; includes a “Do nothing” node that fans out to multiple notification nodes.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry + Image URL Detection
**Overview:** Receives Shopify “new product” event and extracts/detects image link structure so later nodes can fetch the image and post it.  
**Nodes involved:** `Shopify Trigger`, `Code (image link detect)`, `If (separate image link)`

#### Node: Shopify Trigger
- **Type / role:** `shopifyTrigger` — webhook-based trigger for Shopify events (likely “product created”).
- **Configuration (interpreted):** Parameters are empty in JSON; must be configured to listen to the desired Shopify event (e.g., Product created).
- **Inputs/Outputs:** Entry node → outputs to `Code (image link detect)`.
- **Version:** typeVersion `1`.
- **Potential failures / edge cases:**
  - Shopify webhook not registered or not reachable (n8n URL, firewall, SSL).
  - Missing Shopify credentials / insufficient scopes.
  - Event payload differences (image fields may be absent for some products).

#### Node: Code (image link detect)
- **Type / role:** `code` — custom JavaScript to detect or normalize image link(s).
- **Configuration:** Empty parameters (code not provided in JSON). This is a critical missing piece: logic must be implemented to output a predictable field such as `imageUrl` (string) or a list of image URLs.
- **Inputs/Outputs:** From `Shopify Trigger` → to `If (separate image link)`.
- **Version:** typeVersion `2`.
- **Potential failures / edge cases:**
  - Runtime errors if expected fields (e.g., `images[0].src`) are missing.
  - Output schema mismatch causing the downstream IF to misroute.

#### Node: If (separate image link)
- **Type / role:** `if` — branching based on whether image link(s) require separation (single vs multiple, or embedded HTML vs direct URL).
- **Configuration:** Empty; must define boolean condition(s) (e.g., “is array”, “contains comma”, “contains whitespace”, etc.).
- **Outputs / routing:**
  - **True branch** → `Loop Over Items`
  - **False branch** → `Rapiwa (Send Whatsapp message)` and `Send a Notification mail` (this appears to act as an alert/fallback path when image extraction fails or is not in expected format).
- **Version:** typeVersion `2.2`.
- **Potential failures / edge cases:**
  - Incorrect condition causing either over-looping or skipping main publishing path.
  - If condition relies on fields not created due to missing Code node logic.

---

### Block 2 — Looping + Product URL Creation
**Overview:** When multiple items (e.g., multiple images) are present, iterates through them and then builds a public product URL to use in posts.  
**Nodes involved:** `Loop Over Items`, `Create Product URL`

#### Node: Loop Over Items
- **Type / role:** `splitInBatches` — loops items in batches.
- **Configuration:** Empty; must set batch size and looping behavior.
- **Connections:**
  - Has **two outputs** configured in connections:
    - Output 0: **not used** (`[]`)
    - Output 1: goes to `Create Product URL`
- **Version:** typeVersion `3`.
- **Potential failures / edge cases:**
  - Misconfigured output usage: in n8n, Split In Batches typically uses:
    - Output 0 = “Items” for the current batch
    - Output 1 = “No Items Left”
    Here, the workflow routes **Output 1** to `Create Product URL`, which usually means *after* looping completes. If you intended per-item processing, you likely need Output 0 connected instead.
  - Infinite loops if the “continue” mechanism is miswired (not visible here).

#### Node: Create Product URL
- **Type / role:** `set` — constructs/sets fields like product URL, title, etc.
- **Configuration:** Empty; must map Shopify product handle/ID into a URL (e.g., `https://yourstore.com/products/{{handle}}`).
- **Inputs/Outputs:** From `Loop Over Items` → to `Converts product descriptions (HTML) into short`.
- **Version:** typeVersion `3.4`.
- **Potential failures / edge cases:**
  - Shopify payload may not include `handle` depending on event configuration.
  - Store domain may differ per environment; consider storing base URL in an environment variable.

---

### Block 3 — AI Processing (OpenAI + Shopify tool)
**Overview:** Uses OpenAI (LangChain node) to shorten or transform the Shopify HTML description into a concise marketing snippet, potentially using Shopify product data as a tool input.  
**Nodes involved:** `Converts product descriptions (HTML) into short`, `Get a product in Shopify`

#### Node: Converts product descriptions (HTML) into short
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI text generation / transformation via LangChain integration.
- **Configuration:** Empty; must set:
  - Model (e.g., GPT-4.1 mini / GPT-4o mini depending on availability)
  - Prompt/instructions (e.g., “Convert HTML description into 1–2 sentence social post + hashtags”)
  - Input mapping (product description HTML)
- **Connections:**
  - **Main output** → `Get Image form Link`
  - **AI tool input**: receives an `ai_tool` connection from `Get a product in Shopify` (see below).
- **Version:** typeVersion `1.8`.
- **Potential failures / edge cases:**
  - Missing OpenAI credentials or quota limits.
  - Output text too long for some platforms (Instagram caption limits, Telegram, LinkedIn).
  - HTML not stripped correctly if prompt is weak; consider pre-stripping HTML.

#### Node: Get a product in Shopify
- **Type / role:** `shopifyTool` — provides Shopify product retrieval as an AI Tool (LangChain tool integration).
- **Configuration:** Empty; must specify which product to fetch (likely from Trigger product ID).
- **Connections:** Its `ai_tool` output connects into the OpenAI node’s `ai_tool` input.
- **Version:** typeVersion `1`.
- **Potential failures / edge cases:**
  - Wrong product identifier mapping.
  - Shopify API scope issues and rate limits.
  - Tool call failures causing the OpenAI node to fail or produce incomplete output.

---

### Block 4 — Fetch Image + Multi-Platform Posting
**Overview:** Downloads an image from the selected URL and then fans out to multiple posting nodes (WordPress, Facebook, Telegram, LinkedIn, Discord, Instagram pipeline).  
**Nodes involved:** `Get Image form Link`, `Create WordPress Post`, `Post on Facebook page`, `Post on  telegram Channel`, `Post on LinkkedIN Profile`, `Post on LinkedN page`, `Post on Discord Channel`, `Get Image ID for Instagram`, `Publish Post to Instagram`, `Do nothing`

#### Node: Get Image form Link
- **Type / role:** `httpRequest` — fetches image binary from URL.
- **Configuration:** Empty; must set:
  - URL (expression pointing to extracted `imageUrl`)
  - Response: binary file
- **Outputs:** Fans out to:
  - `Create WordPress Post`
  - `Post on Facebook page`
  - `Post on  telegram Channel`
  - `Post on LinkkedIN Profile`
  - `Post on LinkedN page`
  - `Post on Discord Channel`
  - `Do nothing`
  - `Get Image ID for Instagram`
- **Version:** typeVersion `4.2`.
- **Potential failures / edge cases:**
  - 403/404 if image URL private or expiring.
  - Large image causing memory/timeouts.
  - Wrong “response format” (must be binary for downstream image use).

#### Node: Create WordPress Post
- **Type / role:** `wordpress` — creates a new WordPress post (likely product announcement).
- **Configuration:** Empty; must set:
  - Operation: Create post
  - Title, content (from Shopify + OpenAI text), status (draft/publish)
- **Input/Output:** From `Get Image form Link` → to `download image` (used for featured image pipeline).
- **Version:** typeVersion `1`.
- **Potential failures / edge cases:**
  - WordPress authentication issues (application password, OAuth, JWT—depends on node setup).
  - Missing required fields (title/content).
  - Post created but media steps fail, leaving post without featured image.

#### Node: Post on Facebook page
- **Type / role:** `facebookGraphApi` — posts content to a Facebook Page.
- **Configuration:** Empty; must set page ID, message, attached media if supported.
- **Input:** From `Get Image form Link`.
- **Version:** typeVersion `1`.
- **Potential failures / edge cases:** Token expiration, missing permissions (`pages_manage_posts`, etc.), Facebook API restrictions.

#### Node: Post on  telegram Channel
- **Type / role:** `telegram` — posts message/photo to a Telegram channel.
- **Configuration:** Empty; must set chat/channel ID and message/media.
- **Input:** From `Get Image form Link`.
- **Version:** typeVersion `1.2`.
- **Potential failures:** Bot not admin in channel, wrong chat ID, file size limits.

#### Node: Post on LinkkedIN Profile
- **Type / role:** `linkedIn` — posts to a LinkedIn personal profile.
- **Configuration:** Empty; must configure author, text, media.
- **Input:** From `Get Image form Link`.
- **Version:** typeVersion `1`.
- **Potential failures:** LinkedIn API access limitations, token scopes, media upload requirements.

#### Node: Post on LinkedN page
- **Type / role:** `linkedIn` — posts to a LinkedIn company page.
- **Configuration:** Empty; must set organization/page URN and permissions.
- **Input:** From `Get Image form Link`.
- **Version:** typeVersion `1`.
- **Potential failures:** Missing admin rights, organization posting restrictions.

#### Node: Post on Discord Channel
- **Type / role:** `discord` — posts message to a Discord channel.
- **Configuration:** Empty; likely requires bot token and channel ID; can include attachments or embed links.
- **Input:** From `Get Image form Link`.
- **Version:** typeVersion `2`.
- **Potential failures:** Missing permissions, file upload limits, rate limiting.

#### Node: Get Image ID for Instagram
- **Type / role:** `httpRequest` — typically creates an Instagram media container via Graph API using an image URL (step 1).
- **Configuration:** Empty; must call `/{ig-user-id}/media?image_url=...&caption=...`.
- **Input/Output:** From `Get Image form Link` → to `Publish Post to Instagram`.
- **Version:** typeVersion `4.2`.
- **Potential failures:** Instagram requires a **publicly accessible** image URL (not binary upload), container creation errors, business account requirements.

#### Node: Publish Post to Instagram
- **Type / role:** `facebookGraphApi` — publishes the created media container (step 2).
- **Configuration:** Empty; must call `/{ig-user-id}/media_publish?creation_id=...`.
- **Input:** From `Get Image ID for Instagram`.
- **Version:** typeVersion `1`.
- **Potential failures:** Publishing too soon before container is ready; requires polling or wait in some cases.

#### Node: Do nothing
- **Type / role:** `noOp` — used here as a fan-out hub to multiple notification nodes (despite its name).
- **Input/Output:** From `Get Image form Link` → outputs to all notification nodes:
  - `Send a Notification` (Telegram)
  - `Send a Notification1` (Gmail)
  - `Sent a Notification` (Discord)
  - `Rapiwa (sent whatsapp Notification)`
  - `Create message` (Teams)
- **Version:** typeVersion `1`.
- **Potential failures:** None by itself; downstream failures propagate per node.

---

### Block 5 — WordPress Featured Image Pipeline
**Overview:** After creating the WP post, the workflow downloads/edits the image and uploads it to WordPress, then sets it as the post’s featured image.  
**Nodes involved:** `download image`, `Edit Image`, `upload media to wp`, `upload image to meta data`, `set featured image`

#### Node: download image
- **Type / role:** `httpRequest` — downloads image (binary) for editing/uploading to WordPress.
- **Configuration:** Empty; must set URL and binary response.
- **Connections:** From `Create WordPress Post` → `Edit Image`.
- **Version:** typeVersion `4.2`.
- **Edge cases:** Large images, invalid URL, not returning binary.

#### Node: Edit Image
- **Type / role:** `editImage` — transforms image (resize/crop/convert).
- **Configuration:** Empty; should define target size/format suited for WP featured images.
- **Connections:** → `upload media to wp`
- **Version:** typeVersion `1`.
- **Edge cases:** Unsupported input format, missing binary property name.

#### Node: upload media to wp
- **Type / role:** `httpRequest` — custom HTTP request to WordPress media endpoint (`/wp-json/wp/v2/media`) to upload the image.
- **Configuration:** Empty; must set:
  - Method: POST
  - Auth headers (Basic with application password or bearer token)
  - `Content-Disposition` filename
  - Binary body
- **Connections:** → `upload image to meta data`
- **Version:** typeVersion `4.2`.
- **Edge cases:** 401/403 auth, wrong content-type, WP file size limits.

#### Node: upload image to meta data
- **Type / role:** `httpRequest` — likely updates attachment metadata (alt text, title, caption).
- **Configuration:** Empty; typically `POST /wp-json/wp/v2/media/{id}` with JSON body.
- **Connections:** → `set featured image`
- **Version:** typeVersion `4.2`.
- **Edge cases:** Missing media ID from previous step; WP REST permissions.

#### Node: set featured image
- **Type / role:** `httpRequest` — likely updates the post to set `featured_media` to the uploaded media ID.
- **Configuration:** Empty; typically `POST /wp-json/wp/v2/posts/{postId}`.
- **Version:** typeVersion `4.2`.
- **Edge cases:** Missing post ID from `Create WordPress Post`, race condition if post not fully created, permissions.

---

### Block 6 — Notifications / Alerts
**Overview:** Sends alerts/notifications across channels. One branch is used as a fallback from the IF node; another is a fan-out from “Do nothing”.  
**Nodes involved:** `Rapiwa (Send Whatsapp message)`, `Send a Notification mail`, `Send a Notification`, `Send a Notification1`, `Sent a Notification`, `Rapiwa (sent whatsapp Notification)`, `Create message`

#### Node: Rapiwa (Send Whatsapp message)
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — WhatsApp sending via Rapiwa integration.
- **Triggered by:** False branch of `If (separate image link)`.
- **Configuration:** Empty; must set recipient and message.
- **Version:** typeVersion `1`.
- **Edge cases:** Credential issues, invalid phone formatting, provider downtime.

#### Node: Send a Notification mail
- **Type / role:** `gmail` — sends email notification.
- **Triggered by:** False branch of `If (separate image link)`.
- **Configuration:** Empty; must set to/from/subject/body.
- **Version:** typeVersion `2.1`.
- **Edge cases:** OAuth issues, Gmail sending limits.

#### Node: Send a Notification
- **Type / role:** `telegram` — notification message to Telegram.
- **Triggered by:** `Do nothing`.
- **Version:** typeVersion `1.2`.

#### Node: Send a Notification1
- **Type / role:** `gmail` — email notification.
- **Triggered by:** `Do nothing`.
- **Version:** typeVersion `2.1`.

#### Node: Sent a Notification
- **Type / role:** `discord` — notification message to Discord.
- **Triggered by:** `Do nothing`.
- **Version:** typeVersion `2`.

#### Node: Rapiwa (sent whatsapp Notification)
- **Type / role:** `rapiwa` — WhatsApp notification.
- **Triggered by:** `Do nothing`.
- **Version:** typeVersion `1`.

#### Node: Create message
- **Type / role:** `microsoftTeams` — posts a message to a Teams channel/chat.
- **Triggered by:** `Do nothing`.
- **Version:** typeVersion `2`.
- **Edge cases:** Webhook/connector permissions, message formatting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Trigger | n8n-nodes-base.shopifyTrigger | Entry point: receives new product events | — | Code (image link detect) |  |
| Code (image link detect) | n8n-nodes-base.code | Detect/normalize image URL(s) | Shopify Trigger | If (separate image link) |  |
| If (separate image link) | n8n-nodes-base.if | Branch on image URL format/availability | Code (image link detect) | Loop Over Items; Rapiwa (Send Whatsapp message); Send a Notification mail |  |
| Loop Over Items | n8n-nodes-base.splitInBatches | Optional batching/looping | If (separate image link) | Create Product URL (via output index 1) |  |
| Create Product URL | n8n-nodes-base.set | Build product URL + set fields | Loop Over Items | Converts product descriptions (HTML) into short |  |
| Converts product descriptions (HTML) into short | @n8n/n8n-nodes-langchain.openAi | AI short description generation | Create Product URL; (ai_tool) Get a product in Shopify | Get Image form Link |  |
| Get a product in Shopify | n8n-nodes-base.shopifyTool | AI tool: fetch Shopify product data | — | (ai_tool) Converts product descriptions (HTML) into short |  |
| Get Image form Link | n8n-nodes-base.httpRequest | Download/fetch image from URL (binary) | Converts product descriptions (HTML) into short | Create WordPress Post; Post on Facebook page; Post on  telegram Channel; Post on LinkkedIN Profile; Post on LinkedN page; Post on Discord Channel; Do nothing; Get Image ID for Instagram |  |
| Create WordPress Post | n8n-nodes-base.wordpress | Create WP post | Get Image form Link | download image |  |
| download image | n8n-nodes-base.httpRequest | Download image for WP media pipeline | Create WordPress Post | Edit Image |  |
| Edit Image | n8n-nodes-base.editImage | Resize/transform image | download image | upload media to wp |  |
| upload media to wp | n8n-nodes-base.httpRequest | Upload image to WordPress Media Library | Edit Image | upload image to meta data |  |
| upload image to meta data | n8n-nodes-base.httpRequest | Update media metadata (alt/title/etc.) | upload media to wp | set featured image |  |
| set featured image | n8n-nodes-base.httpRequest | Set WP featured image on created post | upload image to meta data | — |  |
| Post on Facebook page | n8n-nodes-base.facebookGraphApi | Post to Facebook Page | Get Image form Link | — |  |
| Get Image ID for Instagram | n8n-nodes-base.httpRequest | Create IG media container (Graph API) | Get Image form Link | Publish Post to Instagram |  |
| Publish Post to Instagram | n8n-nodes-base.facebookGraphApi | Publish IG media container | Get Image ID for Instagram | — |  |
| Post on  telegram Channel | n8n-nodes-base.telegram | Post to Telegram channel | Get Image form Link | — |  |
| Post on LinkkedIN Profile | n8n-nodes-base.linkedIn | Post to LinkedIn profile | Get Image form Link | — |  |
| Post on LinkedN page | n8n-nodes-base.linkedIn | Post to LinkedIn organization page | Get Image form Link | — |  |
| Post on Discord Channel | n8n-nodes-base.discord | Post to Discord channel | Get Image form Link | — |  |
| Do nothing | n8n-nodes-base.noOp | Fan-out hub to notification nodes | Get Image form Link | Send a Notification; Send a Notification1; Sent a Notification; Rapiwa (sent whatsapp Notification); Create message |  |
| Send a Notification | n8n-nodes-base.telegram | Telegram notification | Do nothing | — |  |
| Send a Notification1 | n8n-nodes-base.gmail | Email notification | Do nothing | — |  |
| Sent a Notification | n8n-nodes-base.discord | Discord notification | Do nothing | — |  |
| Rapiwa (sent whatsapp Notification) | n8n-nodes-rapiwa.rapiwa | WhatsApp notification | Do nothing | — |  |
| Create message | n8n-nodes-base.microsoftTeams | Teams message notification | Do nothing | — |  |
| Rapiwa (Send Whatsapp message) | n8n-nodes-rapiwa.rapiwa | WhatsApp alert (fallback branch) | If (separate image link) | — |  |
| Send a Notification mail | n8n-nodes-base.gmail | Email alert (fallback branch) | If (separate image link) | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |

> Note: All sticky notes have empty content in the provided JSON, so no sticky note text can be associated.

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Shopify Trigger**
   2. Configure Shopify credentials (Admin API access).
   3. Select event: **Product created** (or equivalent “new product” event).
   4. Save and (later) activate to register webhook.

2) **Add image URL detection logic**
   1. Add node: **Code** named `Code (image link detect)`
   2. Implement JS that:
      - Reads the Shopify trigger payload (product object)
      - Extracts a primary image URL (e.g., `product.image.src` or `product.images[0].src`)
      - Optionally creates an array of image URLs if multiple exist
      - Outputs consistent fields, e.g.:
        - `imageUrl` (string)
        - `imageUrls` (array)
        - `hasMultipleImages` (boolean)
   3. Connect: `Shopify Trigger` → `Code (image link detect)`

3) **Add branching**
   1. Add node: **If** named `If (separate image link)`
   2. Condition example:
      - If `{{$json.imageUrls && $json.imageUrls.length > 1}}` → true
      - Else → false
   3. Connect: `Code (image link detect)` → `If (separate image link)`

4) **(True branch) Loop items**
   1. Add node: **Split In Batches** named `Loop Over Items`
   2. Set batch size (e.g., 1).
   3. Important: decide intended behavior:
      - If you want per-image posting, connect **Output 0** (“Items”) forward.
      - If you only want to continue after processing all, use Output 1 (“No Items Left”).
   4. Connect: `If (separate image link)` (true output) → `Loop Over Items`

5) **Fallback notifications (False branch of IF)**
   1. Add node: **Rapiwa** named `Rapiwa (Send Whatsapp message)`
      - Configure Rapiwa credentials
      - Set recipient and message like “Image link missing/invalid for product {{$json.title}}”
   2. Add node: **Gmail** named `Send a Notification mail`
      - Configure Gmail OAuth2 credentials
      - Set To/Subject/Body with product details and debug payload
   3. Connect: `If (separate image link)` (false output) → both nodes.

6) **Create Product URL**
   1. Add node: **Set** named `Create Product URL`
   2. Add fields such as:
      - `productUrl` = `https://YOUR_STORE_DOMAIN/products/{{$json.handle}}`
      - `shortTitle` or normalized title
   3. Connect: `Loop Over Items` → `Create Product URL` (choose correct output index as noted in step 4)

7) **Configure Shopify AI tool (optional but present)**
   1. Add node: **Shopify Tool** named `Get a product in Shopify`
   2. Configure Shopify credentials and “get product” operation using the product ID from trigger.
   3. Connect its **AI Tool** output to the OpenAI node’s **ai_tool** input (later step).

8) **OpenAI transformation**
   1. Add node: **OpenAI (LangChain)** named `Converts product descriptions (HTML) into short`
   2. Configure OpenAI credentials.
   3. Set:
      - Input text: Shopify product description HTML
      - Instructions: strip HTML, produce short marketing copy + CTA + optional hashtags
      - Output fields (e.g., `caption`, `summary`)
   4. Connect: `Create Product URL` → `Converts...`
   5. Connect: `Get a product in Shopify` (ai_tool) → `Converts...` (ai_tool)

9) **Fetch image**
   1. Add node: **HTTP Request** named `Get Image form Link`
   2. Configure:
      - URL = `{{$json.imageUrl}}` (or the current loop item URL)
      - Response = **File** (binary)
      - Binary property name (e.g., `data`)
   3. Connect: `Converts...` → `Get Image form Link`

10) **WordPress post creation**
   1. Add node: **WordPress** named `Create WordPress Post`
   2. Configure WP credentials.
   3. Operation: Create Post
   4. Map:
      - Title = product title
      - Content = AI summary + product URL + image embed as needed
      - Status = draft/publish
   5. Connect: `Get Image form Link` → `Create WordPress Post`

11) **WordPress featured image pipeline**
   1. Add **HTTP Request** node `download image` (or reuse binary from step 9; current workflow downloads again)
      - URL = same image URL
      - Response = File (binary)
   2. Add **Edit Image** node `Edit Image` (resize/convert)
   3. Add **HTTP Request** node `upload media to wp`
      - POST to `https://YOUR_WP_SITE/wp-json/wp/v2/media`
      - Auth headers
      - Send binary as body
   4. Add **HTTP Request** node `upload image to meta data`
      - POST/PATCH to `/wp-json/wp/v2/media/{mediaId}` with alt text/title
   5. Add **HTTP Request** node `set featured image`
      - POST/PATCH to `/wp-json/wp/v2/posts/{postId}` with `featured_media: {mediaId}`
   6. Connect in order:
      - `Create WordPress Post` → `download image` → `Edit Image` → `upload media to wp` → `upload image to meta data` → `set featured image`

12) **Social posting fan-out**
   1. Add and configure:
      - **Facebook Graph API** node `Post on Facebook page` (page posting)
      - **Telegram** node `Post on  telegram Channel` (send photo/message)
      - **LinkedIn** node `Post on LinkkedIN Profile`
      - **LinkedIn** node `Post on LinkedN page`
      - **Discord** node `Post on Discord Channel`
   2. Connect all of them from: `Get Image form Link` (fan-out).
   3. Ensure each node maps the same caption/summary and includes `productUrl`.

13) **Instagram two-step publishing**
   1. Add **HTTP Request** node `Get Image ID for Instagram`
      - Call IG Graph endpoint to create media container
      - Use **public image URL** (not binary)
   2. Add **Facebook Graph API** node `Publish Post to Instagram`
      - Publish the container using returned `creation_id`
   3. Connect: `Get Image form Link` → `Get Image ID for Instagram` → `Publish Post to Instagram`

14) **Notifications fan-out (“Do nothing”)**
   1. Add **No Op** node named `Do nothing`
   2. Connect: `Get Image form Link` → `Do nothing`
   3. Add notification nodes and connect from `Do nothing`:
      - Telegram `Send a Notification`
      - Gmail `Send a Notification1`
      - Discord `Sent a Notification`
      - Rapiwa `Rapiwa (sent whatsapp Notification)`
      - Microsoft Teams `Create message`

15) **Credentials checklist**
   - Shopify: Admin API + webhook permissions
   - OpenAI: API key
   - WordPress: application password or other supported auth
   - Facebook/Instagram: Graph API token with correct page/IG scopes
   - LinkedIn: OAuth app with posting permissions
   - Telegram: bot token + channel admin rights
   - Discord: bot token + channel permissions
   - Gmail: OAuth2
   - Microsoft Teams: incoming webhook or OAuth (depending on node mode)
   - Rapiwa: provider credentials

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes are present but contain no text in the provided workflow JSON. | n8n Sticky Note nodes: Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5 |
| Many nodes have empty parameters; they must be configured before the workflow can run. | Applies to nearly all nodes in this workflow export |
| The `Split In Batches` node is wired using output index 1, which often corresponds to “No Items Left”. Verify intended loop behavior. | Node: Loop Over Items |

