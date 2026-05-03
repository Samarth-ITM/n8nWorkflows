Translate RSS news and publish to WordPress, Facebook, LinkedIn and Telegram

https://n8nworkflows.xyz/workflows/translate-rss-news-and-publish-to-wordpress--facebook--linkedin-and-telegram-13171


# Translate RSS news and publish to WordPress, Facebook, LinkedIn and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Translate RSS news and publish to WordPress, Facebook, LinkedIn and Telegram  
**Workflow name (JSON):** Automated Translate and Publish Content Worldwide Across Multiple Platforms

**Purpose:**  
This workflow periodically pulls articles from multiple RSS feeds, prepares and translates post content, extracts an image from the article, processes the image (including optional watermark/edit steps), publishes the article to WordPress (with a featured image), and then posts the media/content to multiple social channels (Facebook, Telegram, LinkedIn profile, LinkedIn page). It also sends status/notification messages to several channels (Gmail, Telegram text, Discord, Microsoft Teams, and a custom “Rapiwa” node).

**Logical blocks (based on node dependencies):**
1.1 **Scheduling & Global Settings** → hourly trigger and setting translation language + RSS URLs  
1.2 **RSS Feed Iteration (multi-feed loop)** → iterate RSS feed URLs and fetch items  
1.3 **Per-Article Processing Loop** → iterate through RSS items; prepare fields and translate  
1.4 **Image Discovery & Download** → fetch article page (or content) to extract an image URL; download to binary  
1.5 **WordPress Publishing + Featured Image** → create WP post, download/edit image, upload to WP, set as featured  
1.6 **Image Post Distribution (Facebook/Telegram/LinkedIn)** → watermark then publish to social platforms  
1.7 **Operational Notifications** → send workflow summary/notifications to Gmail/Telegram/Discord/Teams/Rapiwa  
1.8 **Rate limiting / pacing** → wait nodes controlling iteration pacing and avoiding API throttling

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Global Settings

**Overview:**  
Starts the workflow every hour, sets a target translation language code, then provides a list of RSS feed URLs to process.

**Nodes involved:**
- Schedule Trigger (hourly)
- Sets Translate Language Code
- RSS News Sites Urls
- Split Out

#### Node: Schedule Trigger (hourly)
- **Type / role:** `Schedule Trigger` — entry point, runs workflow on a schedule.
- **Configuration (interpreted):** Runs **hourly** (exact minute not shown in JSON due to empty parameters).
- **Connections:** Outputs to **Sets Translate Language Code**.
- **Edge cases:** n8n instance time zone affects “hourly” timing; if instance is paused/offline, executions may be delayed or skipped depending on n8n settings.

#### Node: Sets Translate Language Code
- **Type / role:** `Set` — defines a translation target language code for downstream translation.
- **Configuration:** Not specified in JSON (empty parameters). Intended to set fields like `languageCode` (e.g., `fr`, `es`, `de`).
- **Connections:** Outputs to **RSS News Sites Urls**.
- **Edge cases:** If the language field isn’t set, Google Translate node may fail or default unexpectedly.

#### Node: RSS News Sites Urls
- **Type / role:** `Set` — defines an array/list of RSS feed URLs.
- **Configuration:** Not specified in JSON (empty parameters). Intended to create something like `rssUrls: [ ... ]`.
- **Connections:** Outputs to **Split Out**.
- **Edge cases:** Invalid URL format, empty list, or feeds that block automated clients.

#### Node: Split Out
- **Type / role:** `Split Out` — converts an array of RSS URLs into individual items (one URL per item).
- **Connections:** Outputs to **Loop Over Items**.
- **Edge cases:** If the expected array field doesn’t exist, it may output nothing (no feeds processed).

---

### 2.2 RSS Feed Iteration (multi-feed loop)

**Overview:**  
Loops over each RSS feed URL item, and for each URL fetches the RSS entries.

**Nodes involved:**
- Loop Over Items
- Wait1
- Fetches news articles from the RSS feed URL

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` (v3) — iterates through feed URL items in batches (commonly batch size 1).
- **Configuration:** Not specified in JSON (empty parameters). Default batch behavior applies.
- **Connections:**
  - Main output 0 → **Code (get total workflow)** (for notifications/summary)
  - Main output 1 → **Fetches news articles from the RSS feed URL**
- **Edge cases:** If batch size > 1, downstream nodes might need to handle multiple URLs per run; “empty parameters” means you must confirm defaults in your n8n version.

#### Node: Wait1
- **Type / role:** `Wait` — pacing control between loop iterations (used as a loop-back timing gate).
- **Connections:** Outputs back to **Loop Over Items**.
- **Edge cases:** If configured for “webhook wait” vs “time wait” incorrectly, the workflow can stall. (Parameters empty in JSON; check node UI.)

#### Node: Fetches news articles from the RSS feed URL
- **Type / role:** `RSS Feed Read` — fetches and parses RSS items.
- **Notable setting:** `onError: continueRegularOutput` — workflow continues even if a feed fails.
- **Configuration:** RSS URL is likely taken from the current item (expression), but not visible due to empty parameters.
- **Connections:** Outputs to **Loop Over Items2**.
- **Edge cases:** Feed timeouts, invalid XML, 403/429 blocking, or feeds returning no items.

---

### 2.3 Per-Article Processing Loop (prepare + translate)

**Overview:**  
Iterates each RSS item, extracts relevant fields (title/link/snippet/content), translates content, then proceeds to image extraction.

**Nodes involved:**
- Loop Over Items2
- Code (Prepare Post): Extracted fields – title, link, contentSnippet, content
- Translate to Your Own Language
- HTTP (link to code)

#### Node: Loop Over Items2
- **Type / role:** `Split In Batches` (v3) — iterates through RSS items (articles).
- **Connections:**
  - Output 0 → **Wait1** (loop pacing / continue outer feed loop)
  - Output 1 → **Code (Prepare Post)...**
- **Edge cases:** If items are nested structures, you may need an explicit “Split Out” before batching; otherwise only one item may be processed.

#### Node: Code (Prepare Post): Extracted fields – title, link, contentSnippet, content
- **Type / role:** `Code` (v2) — normalizes the RSS item into a consistent schema for translation/publishing.
- **Configuration:** Not provided; implied behavior from name:
  - Extract fields: `title`, `link`, `contentSnippet`, `content`
  - Possibly build `wpTitle`, `wpContent`, `socialCaption`, etc.
- **Connections:** Outputs to **Translate to Your Own Language**.
- **Edge cases:** RSS feeds differ widely (`content:encoded`, `description`, etc.). Code must handle missing fields or HTML.

#### Node: Translate to Your Own Language
- **Type / role:** `Google Translate` (v2) — translates prepared text into the chosen language.
- **Configuration:** Not shown; expected:
  - Source text: from prepared content fields
  - Target language: from **Sets Translate Language Code**
- **Connections:** Outputs to **HTTP (link to code)**.
- **Edge cases:** API quota limits, large text length limits, HTML translation issues, missing language code.

#### Node: HTTP (link to code)
- **Type / role:** `HTTP Request` (v4.2) — fetches the article page or a link derived from the RSS item to enable image extraction.
- **Configuration:** Not shown; likely GET request to the RSS item `link`.
- **Connections:** Outputs to **Code (get post image link)**.
- **Edge cases:** Websites blocking bots, redirects, Cloudflare challenges, timeouts; HTML parsing in downstream code can break if page structure changes.

---

### 2.4 Image Discovery & Download

**Overview:**  
Extracts a representative image URL for the article, then downloads that image into binary data for upload and image processing.

**Nodes involved:**
- Code (get post image link)
- HTTP (image link to binary data)

#### Node: Code (get post image link)
- **Type / role:** `Code` (v2) — parses fetched HTML/content and selects an image URL (e.g., og:image or first `<img>`).
- **Configuration:** Not shown; expected output fields:
  - `imageUrl` (string)
- **Connections:** Outputs to **HTTP (image link to binary data)**.
- **Edge cases:** No image found, relative URLs, lazy-loaded images, blocked content; must handle invalid/empty URL.

#### Node: HTTP (image link to binary data)
- **Type / role:** `HTTP Request` (v4.2) — downloads the image as binary.
- **Configuration:** Likely:
  - URL: `{{$json.imageUrl}}`
  - Response: binary (set “Download” / “Response Format: File”)
- **Connections:** Outputs to:
  - **Create WordPress Post**
  - **Add Watermark**
- **Edge cases:** Large images, unsupported formats, 403 hotlink protection, wrong MIME type; binary property name must match what downstream expects.

---

### 2.5 WordPress Publishing + Featured Image

**Overview:**  
Creates a WordPress post, downloads and edits image (for WP media), uploads it to WordPress, attaches metadata, then sets it as featured image.

**Nodes involved:**
- Create WordPress Post
- HTTP (image link to binary data)1
- Edit Image
- upload media to wp
- upload image to meta data
- set featured image

#### Node: Create WordPress Post
- **Type / role:** `Wordpress` — creates a new post.
- **Configuration:** Not shown; expected:
  - Title/content from translation output
  - Status (draft/publish) depending on desired behavior
- **Connections:** Outputs to **HTTP (image link to binary data)1**.
- **Edge cases:** Auth failures (application password/OAuth), required fields missing, HTML sanitation, duplicate posting (no dedupe node present).

#### Node: HTTP (image link to binary data)1
- **Type / role:** `HTTP Request` (v4.2) — (re)downloads image for WP pipeline or transforms based on WP needs.
- **Configuration:** Not shown; likely downloads from `imageUrl`.
- **Connections:** Outputs to **Edit Image**.
- **Edge cases:** Same as the other image download node; also risk of mismatch between WP post data item and image item if item linking is not consistent.

#### Node: Edit Image
- **Type / role:** `Edit Image` — transforms image (resize/crop/format).
- **Configuration:** Not shown.
- **Connections:** Outputs to **upload media to wp**.
- **Edge cases:** Requires binary input; if binary property name differs, node fails.

#### Node: upload media to wp
- **Type / role:** `HTTP Request` (v4.2) — uploads media to WordPress Media Library (typically `POST /wp-json/wp/v2/media`).
- **Configuration:** Not shown; expected:
  - Multipart/form-data with binary file
  - Auth header (Basic w/ app password or bearer)
- **Connections:** Outputs to **upload image to meta data**.
- **Edge cases:** Wrong endpoint, insufficient permissions, file size limits.

#### Node: upload image to meta data
- **Type / role:** `HTTP Request` (v4.2) — updates uploaded media metadata (title/alt text/caption).
- **Configuration:** Not shown; expected `POST/PATCH /wp-json/wp/v2/media/{id}`.
- **Connections:** Outputs to **set featured image**.
- **Edge cases:** Missing media ID from previous step, permission errors.

#### Node: set featured image
- **Type / role:** `HTTP Request` (v4.2) — sets the created post’s `featured_media` to the uploaded media ID (typically update post endpoint).
- **Configuration:** Not shown; expected `POST/PATCH /wp-json/wp/v2/posts/{postId}` with JSON body.
- **Connections:** None shown in connections (terminal for WP path).
- **Edge cases:** Post ID not available, wrong field name, REST API disabled by WP security plugins.

---

### 2.6 Image Post Distribution (Facebook/Telegram/LinkedIn)

**Overview:**  
Adds a watermark to the downloaded image and posts it to Facebook, Telegram, and LinkedIn (profile and page). Includes a wait step after the LinkedIn page post.

**Nodes involved:**
- Add Watermark
- Facebook media post
- Telegram Image post
- Create profile image post
- Create page image post
- Wait 2 second

#### Node: Add Watermark
- **Type / role:** `Edit Image` — overlays watermark or otherwise modifies image for social posting.
- **Configuration:** Not shown; despite name, the actual watermark settings must be configured in node UI.
- **Connections:** Outputs in parallel to:
  - **Facebook media post**
  - **Telegram Image post**
  - **Create profile image post**
  - **Create page image post**
- **Edge cases:** If watermark asset not available, or binary field mismatch; image formatting may break some APIs.

#### Node: Facebook media post
- **Type / role:** `Facebook Graph API` — publishes an image/media post.
- **Configuration:** Not shown; expected:
  - Page ID / target
  - Message/caption
  - Media upload from binary
- **Connections:** Terminal.
- **Edge cases:** Token expiration, missing permissions (`pages_manage_posts`, etc.), rate limits.

#### Node: Telegram Image post
- **Type / role:** `Telegram` (v1.2) — sends a photo message to a chat/channel.
- **Configuration:** Not shown; expected:
  - Chat ID
  - Photo from binary
  - Caption from translated content
- **Connections:** Terminal.
- **Edge cases:** Bot not admin in channel, file size limit, HTML/Markdown parse issues.

#### Node: Create profile image post
- **Type / role:** `LinkedIn` — creates an image post on a personal profile.
- **Configuration:** Not shown; expected OAuth2 credentials and author URN.
- **Connections:** Terminal.
- **Edge cases:** LinkedIn API permission issues, required “organization vs member” author mismatch.

#### Node: Create page image post
- **Type / role:** `LinkedIn` — creates an image post on a LinkedIn page (organization).
- **Configuration:** Not shown; expected organization URN and proper permissions.
- **Connections:** Outputs to **Wait 2 second**.
- **Edge cases:** LinkedIn rate limiting; page posting requires additional permissions and correct org admin rights.

#### Node: Wait 2 second
- **Type / role:** `Wait` — pauses briefly before continuing the per-article loop.
- **Configuration:** Intended to wait ~2 seconds (name), but parameters are empty in JSON; must be set in UI.
- **Connections:** Outputs to **Loop Over Items2** (continues processing next article).
- **Edge cases:** If configured as “wait for webhook”, execution will stall.

---

### 2.7 Operational Notifications

**Overview:**  
Generates a “total workflow” or summary payload and sends it to multiple notification channels.

**Nodes involved:**
- Code (get total workflow)
- Send a message2 (Gmail)
- Send a text message (Telegram)
- Rapiwa
- Send a message (Discord)
- Create message (Microsoft Teams)

#### Node: Code (get total workflow)
- **Type / role:** `Code` (v2) — composes a status message/summary (likely including counts, current feed URL, timestamps).
- **Connections:** Fan-out to:
  - **Send a message2** (Gmail)
  - **Send a text message** (Telegram)
  - **Rapiwa**
  - **Send a message** (Discord)
  - **Create message** (Microsoft Teams)
- **Edge cases:** If it assumes certain fields exist (like total items), it may fail depending on upstream batch context.

#### Node: Send a message2
- **Type / role:** `Gmail` (v2.1) — sends an email notification.
- **Configuration:** Not shown; requires Gmail OAuth2 or service account (depending on n8n Gmail node mode).
- **Edge cases:** Token expiration, Gmail sending limits.

#### Node: Send a text message
- **Type / role:** `Telegram` (v1.2) — sends a text message (status).
- **Edge cases:** Same as other Telegram node; chat ID/permissions.

#### Node: Rapiwa
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — custom community node (unknown exact API).
- **Configuration:** Not shown.
- **Edge cases:** Node availability (must be installed), breaking changes on update, credential requirements unknown.

#### Node: Send a message
- **Type / role:** `Discord` (v2) — sends a message (likely to a channel via webhook or bot).
- **Edge cases:** Webhook invalid/rotated, rate limits.

#### Node: Create message
- **Type / role:** `Microsoft Teams` (v2) — posts a Teams message (channel/chat).
- **Edge cases:** Auth and tenant policy restrictions; connector/webhook configuration.

---

### 2.8 Sticky Notes (documentation nodes)

**Overview:**  
There are multiple sticky note nodes, but their `content` fields are empty in the JSON provided. They still count as nodes in the workflow canvas but do not affect execution.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note3
- Sticky Note6
- Sticky Note8

**Edge cases:** None (non-executing UI elements).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger (hourly) | Schedule Trigger | Starts workflow hourly | — | Sets Translate Language Code |  |
| Sets Translate Language Code | Set | Defines target translation language | Schedule Trigger (hourly) | RSS News Sites Urls |  |
| RSS News Sites Urls | Set | Defines list of RSS feed URLs | Sets Translate Language Code | Split Out |  |
| Split Out | Split Out | Splits RSS URL list into items | RSS News Sites Urls | Loop Over Items |  |
| Loop Over Items | Split In Batches (v3) | Iterates over RSS feed URLs | Split Out, Wait1 | Code (get total workflow), Fetches news articles from the RSS feed URL |  |
| Code (get total workflow) | Code (v2) | Builds execution summary/status message | Loop Over Items | Send a message2; Send a text message; Rapiwa; Send a message; Create message |  |
| Send a message2 | Gmail (v2.1) | Email notification | Code (get total workflow) | — |  |
| Send a text message | Telegram (v1.2) | Telegram text notification | Code (get total workflow) | — |  |
| Rapiwa | rapiwa | Custom notification/integration | Code (get total workflow) | — |  |
| Send a message | Discord (v2) | Discord notification | Code (get total workflow) | — |  |
| Create message | Microsoft Teams (v2) | Teams notification | Code (get total workflow) | — |  |
| Fetches news articles from the RSS feed URL | RSS Feed Read (v1.2) | Fetch RSS items for a feed URL | Loop Over Items | Loop Over Items2 |  |
| Loop Over Items2 | Split In Batches (v3) | Iterates through RSS articles | Fetches news articles from the RSS feed URL, Wait 2 second | Wait1, Code (Prepare Post): Extracted fields – title, link, contentSnippet, content |  |
| Code (Prepare Post): Extracted fields – title, link, contentSnippet, content | Code (v2) | Normalizes article fields for translation | Loop Over Items2 | Translate to Your Own Language |  |
| Translate to Your Own Language | Google Translate (v2) | Translates prepared text | Code (Prepare Post)... | HTTP (link to code) |  |
| HTTP (link to code) | HTTP Request (v4.2) | Fetches article page/content for image extraction | Translate to Your Own Language | Code (get post image link) |  |
| Code (get post image link) | Code (v2) | Extracts an image URL from page/content | HTTP (link to code) | HTTP (image link to binary data) |  |
| HTTP (image link to binary data) | HTTP Request (v4.2) | Downloads article image as binary | Code (get post image link) | Create WordPress Post; Add Watermark |  |
| Create WordPress Post | WordPress (v1) | Creates WP post | HTTP (image link to binary data) | HTTP (image link to binary data)1 |  |
| HTTP (image link to binary data)1 | HTTP Request (v4.2) | Downloads image for WP pipeline | Create WordPress Post | Edit Image |  |
| Edit Image | Edit Image (v1) | Resizes/edits image for WP media | HTTP (image link to binary data)1 | upload media to wp |  |
| upload media to wp | HTTP Request (v4.2) | Uploads image to WP media endpoint | Edit Image | upload image to meta data |  |
| upload image to meta data | HTTP Request (v4.2) | Updates WP media metadata | upload media to wp | set featured image |  |
| set featured image | HTTP Request (v4.2) | Sets WP post featured image | upload image to meta data | — |  |
| Add Watermark | Edit Image (v1) | Adds watermark for social posting | HTTP (image link to binary data) | Facebook media post; Telegram Image post; Create profile image post; Create page image post |  |
| Facebook media post | Facebook Graph API (v1) | Publishes image post to Facebook | Add Watermark | — |  |
| Telegram Image post | Telegram (v1.2) | Publishes image post to Telegram | Add Watermark | — |  |
| Create profile image post | LinkedIn (v1) | Publishes image post to LinkedIn profile | Add Watermark | — |  |
| Create page image post | LinkedIn (v1) | Publishes image post to LinkedIn page | Add Watermark | Wait 2 second |  |
| Wait 2 second | Wait (v1.1) | Short delay before continuing article loop | Create page image post | Loop Over Items2 |  |
| Wait1 | Wait (v1.1) | Loop pacing for feed/article iteration | Loop Over Items2 | Loop Over Items |  |
| Sticky Note | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note3 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note6 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note8 | Sticky Note | Canvas annotation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
1. Add **Schedule Trigger** node.
2. Set it to run **every 1 hour** (hourly).

2) **Set translation language**
3. Add a **Set** node named **Sets Translate Language Code**.
4. Add a field such as:
   - `targetLang` = e.g. `fr` (choose your target)
5. Connect: **Schedule Trigger → Sets Translate Language Code**

3) **Provide RSS feed URLs**
6. Add a **Set** node named **RSS News Sites Urls**.
7. Create an array field, for example:
   - `rssUrls` = `[ "https://example.com/feed", "https://example.org/rss" ]`
8. Connect: **Sets Translate Language Code → RSS News Sites Urls**

4) **Split URL list into items**
9. Add **Split Out** node.
10. Configure it to split the `rssUrls` array into separate items (one URL per item).
11. Connect: **RSS News Sites Urls → Split Out**

5) **Loop over feed URLs**
12. Add **Split In Batches** named **Loop Over Items** (batch size typically 1).
13. Connect: **Split Out → Loop Over Items**
14. Add **Wait** named **Wait1** (configure as a time-based wait, e.g., 1–2 seconds or as needed).
15. Connect: **Wait1 → Loop Over Items** (loop-back)

6) **Fetch RSS items**
16. Add **RSS Feed Read** named **Fetches news articles from the RSS feed URL**.
17. Set its RSS URL to an expression using the current item from the URL loop, e.g. `{{$json.rssUrl}}` or the field created by Split Out.
18. Set **Error Handling** to continue (equivalent to `continueRegularOutput`).
19. Connect: **Loop Over Items (second output) → RSS Feed Read**

7) **Loop over articles**
20. Add **Split In Batches** named **Loop Over Items2** (batch size 1).
21. Connect: **RSS Feed Read → Loop Over Items2**
22. Connect **Loop Over Items2 (first output)** to **Wait1** (to coordinate pacing/outer loop as in the original).

8) **Prepare post fields**
23. Add **Code** node named **Code (Prepare Post): Extracted fields – title, link, contentSnippet, content**.
24. In code, map RSS item fields into consistent keys, e.g.:
   - `title`, `link`, `contentSnippet`, `content`
   - build `postBody` for WP/social
25. Connect: **Loop Over Items2 (second output) → Code (Prepare Post)...**

9) **Translate**
26. Add **Google Translate** node named **Translate to Your Own Language**.
27. Configure:
   - Text to translate: prepared fields (e.g. `{{$json.content}}` or assembled text)
   - Target language: `{{$json.targetLang}}` (from earlier Set; ensure it is carried forward or merged)
28. Connect: **Code (Prepare Post)... → Translate**

10) **Fetch article page and extract image**
29. Add **HTTP Request** named **HTTP (link to code)**, method GET.
30. URL: use the article link (e.g. `{{$json.link}}`).
31. Add **Code** node named **Code (get post image link)** to parse the HTTP response and output `imageUrl`.
32. Connect: **Translate → HTTP (link to code) → Code (get post image link)**

11) **Download image to binary**
33. Add **HTTP Request** named **HTTP (image link to binary data)**.
34. Set URL to `{{$json.imageUrl}}`.
35. Enable “Download” / “Response as file” so output is **binary**.
36. Connect: **Code (get post image link) → HTTP (image link to binary data)**

12) **Create WordPress post**
37. Add **WordPress** node named **Create WordPress Post**.
38. Configure credentials (WordPress API):
   - Base URL of site
   - Auth (commonly Application Password)
39. Set post fields using translated content:
   - Title, Content, Status, Categories/Tags as needed
40. Connect: **HTTP (image link to binary data) → Create WordPress Post**

13) **WP image pipeline (download/edit/upload/set featured)**
41. Add **HTTP Request** named **HTTP (image link to binary data)1** to download image again (or reuse existing binary; the original uses a second node).
42. Connect: **Create WordPress Post → HTTP (image link to binary data)1**
43. Add **Edit Image** node named **Edit Image** (resize/format).
44. Connect: **HTTP (image link to binary data)1 → Edit Image**
45. Add **HTTP Request** node named **upload media to wp** to `POST /wp-json/wp/v2/media` with binary file.
46. Add **HTTP Request** node named **upload image to meta data** to update media details.
47. Add **HTTP Request** node named **set featured image** to update the created post with `featured_media`.
48. Connect chain: **Edit Image → upload media to wp → upload image to meta data → set featured image**

14) **Watermark + social posting**
49. Add **Edit Image** node named **Add Watermark**; configure watermark overlay.
50. Connect: **HTTP (image link to binary data) → Add Watermark**
51. Add social nodes and connect from **Add Watermark** in parallel:
   - **Facebook Graph API** node “Facebook media post” (configure Page, caption, permissions/credentials)
   - **Telegram** node “Telegram Image post” (bot token + chat id, send photo)
   - **LinkedIn** node “Create profile image post” (OAuth2, author = member)
   - **LinkedIn** node “Create page image post” (OAuth2, author = organization)
52. Add **Wait** node “Wait 2 second” configured as a 2-second delay.
53. Connect: **Create page image post → Wait 2 second → Loop Over Items2** (loop continuation)

15) **Notifications block**
54. Add **Code** node “Code (get total workflow)” connected from **Loop Over Items** first output.
55. Add and configure notification nodes:
   - Gmail “Send a message2”
   - Telegram “Send a text message”
   - Discord “Send a message”
   - Microsoft Teams “Create message”
   - Custom node “Rapiwa” (install community node first)
56. Connect fan-out: **Code (get total workflow) → each notification node**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist in the workflow but contain no text in the provided JSON. | Canvas documentation nodes: Sticky Note, Sticky Note1, Sticky Note3, Sticky Note6, Sticky Note8 |
| Multiple nodes have empty parameters in the JSON; key configuration (RSS URL field, translation fields, WP endpoints, social captions) must be set in the n8n UI to match your environment. | Applies to most integration nodes and HTTP Requests |
| The workflow uses a community/custom node `n8n-nodes-rapiwa.rapiwa`; it must be installed on the n8n instance before import/execution. | Node: Rapiwa (custom) |

