Create WooCommerce products via Telegram bot with Google Gemini AI

https://n8nworkflows.xyz/workflows/create-woocommerce-products-via-telegram-bot-with-google-gemini-ai-12326


# Create WooCommerce products via Telegram bot with Google Gemini AI

## 1. Workflow Overview

**Workflow title (given):** Create WooCommerce products via Telegram bot with Google Gemini AI  
**Workflow name (in JSON):** Telegram Bot Based WooCommerce Product Creator

**Purpose:**  
A Telegram bot conducts a step-by-step conversation to collect product info (name, images, optional features, pricing), generates SEO copy (description, short description, slug) and image metadata (title + alt tags) using **Google Gemini** via LangChain nodes, then creates a **draft WooCommerce product** via the WooCommerce REST API. It stores conversation state and temporary image data in **n8n Data Tables** and supports an **/abort** command to reset state.

### 1.1 Entry & Session State
- Trigger on Telegram messages and callback queries.
- Fetch existing session row for the user (`chat_id`) from a Data Table.

### 1.2 Abort Handling (/abort)
- If user sends `/abort` at any time, delete session state and image rows and notify user.

### 1.3 Conversation State Router
- If user has an active session, route to the appropriate step using a Switch based on `current_step`.
- If no session is active, prompt the user to type `/start`.

### 1.4 Data Collection Steps
- `/start` → ask product name → ask images (upload + Done button) → ask features (text or None) → ask regular price → ask sale price (text or Same button) → show a review summary and request approval.

### 1.5 Approval & Product Creation
- On approval, generate:
  - Product description, short description, slug (two variants depending on whether features were provided).
  - Per-image SEO title and alt tags using AI Agent + structured parser.
- Create product in WooCommerce as **draft** with images array and prices.
- Notify user on Telegram.

### 1.6 Cleanup / Reset
- On completion or abort:
  - Delete session row from product manager table.
  - Delete user image rows from images table.
- Prompt the user to `/start` again.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Telegram Entry + Load Session State
**Overview:** Receives updates from Telegram and loads the existing session (if any) from the WooCommerce Product Manager Data Table.  
**Nodes involved:** `Telegram Trigger`, `Get User State`, `If /abort`, `Delete row(s)2`, `Delete row(s)3`, `Send a text message11`, `Code1`, `If3`, `Get User State1`

#### Telegram Trigger
- **Type/role:** `telegramTrigger` — entry point for messages & callback queries.
- **Config (interpreted):**
  - Updates: `message`, `callback_query`
  - Uses Telegram credentials: **WooCommerce Product Manager** bot.
- **Key expressions:** none (trigger provides payload).
- **Outputs:** to `Get User State`.
- **Failures/edge cases:**
  - Bot token invalid, webhook misconfigured, Telegram connectivity.
  - Callback queries vs messages: downstream nodes frequently branch on `message` vs `callback_query`.

#### Get User State
- **Type/role:** `dataTable` (operation: get) — fetch session row by `chat_id`.
- **Config:**
  - Data table: “WooCommerce Product Manager”
  - Filter: `chat_id = chat.id` (from message or callback_query message)
  - `alwaysOutputData = true` so the flow continues even if no row exists.
- **Outputs:** to `If /abort`.
- **Edge cases:**
  - No matching row → output empty item; later expressions must handle missing fields.

#### If /abort
- **Type/role:** `if` — detects abort command.
- **Condition:** compares message text (or callback data) to `/abort`.
- **True path:** resets tables and informs the user.
- **False path:** proceeds to `Code1` (normal flow).
- **Edge cases:**
  - If a callback query contains `/abort` as data (unlikely but supported).

#### Delete row(s)2
- **Type/role:** `dataTable` deleteRows — deletes session row for this `chat_id` in “WooCommerce Product Manager”.
- **Input:** from `If /abort` true path.
- **Output:** to `Delete row(s)3`.
- **Failure modes:** permission issues, table not found, transient DB errors.

#### Delete row(s)3
- **Type/role:** `dataTable` deleteRows — deletes image rows for this `chat_id` in “User_Images (Woocommerce)”.
- **Config:** `alwaysOutputData = true`
- **Output:** to `Send a text message11`.
- **Edge cases:** no rows to delete (should still succeed).

#### Send a text message11
- **Type/role:** `telegram` — informs user the process was aborted.
- **Config note:** message text set; **chatId is not explicitly set in parameters** (unusual).
  - In many n8n Telegram nodes, `chatId` is required; if omitted, this node may fail unless n8n auto-infers it (typically it does not).
- **Output:** none.
- **Failure modes:** will likely error due to missing chatId unless n8n version/credential behavior supports inference.

#### Code1
- **Type/role:** `code` — passes through all items from `Get User State`.
- **Code:** `return $('Get User State').all();`
- **Purpose:** ensures downstream gets the session record(s) as items.
- **Output:** to `If3`.
- **Edge cases:** if `Get User State` returned nothing, `.all()` may be empty → downstream must handle.

#### If3
- **Type/role:** `if` — checks if a session exists.
- **Condition:** `notEmpty($(Get User State).item.json.chat_id)`
- **True:** session exists → `Get User State1`
- **False:** session not found → `If1` (start gating)
- **Edge cases:** `Get User State` output structure differs when empty; expression may resolve to undefined.

#### Get User State1
- **Type/role:** `dataTable` get (returnAll = true) — loads session row(s) by chat_id again.
- **Output:** to `Switch`
- **Note:** This is redundant with `Get User State` and `Code1` but used to ensure `current_step` is available.
- **Edge cases:** multiple rows for same chat_id (should not happen if unique constraint not enforced).

---

### Block 2.2 — “Start” Gate When No Active Session
**Overview:** When the user is not in an active session, the workflow expects `/start` to begin; otherwise it prompts the user to type `/start`.  
**Nodes involved:** `If1`, `Send a text message3`, `Upsert row(s)`, `Send a text message5`

#### If1
- **Type/role:** `if` — checks whether the incoming message text equals `/start`.
- **Condition:** `Telegram Trigger.message.text == "/start"`
- **True path:** greet and initialize session.
- **False path:** ask user to type /start.
- **Failure modes:** if update is callback query (no message.text), condition may be undefined; but the workflow routes here mainly when no session exists.

#### Send a text message3
- **Type/role:** `telegram` — asks for product name.
- **Text:** “Welcome!... what is the name…”
- **chatId expression:** robustly selects message chat id or callback query message chat id.
- **Output:** to `Upsert row(s)` (state initialization).

#### Upsert row(s)
- **Type/role:** `dataTable` upsert — creates/updates a session with `current_step=awaiting_name`.
- **Columns set:** `chat_id`, `current_step="awaiting_name"`
- **Mapping note:** some schema fields are marked “removed” in this node’s schema definition, but the table itself later uses them—this mismatch can cause confusion if the underlying table schema differs.
- **Edge cases:** if the table lacks a unique key on chat_id, upsert logic may not behave as intended.

#### Send a text message5
- **Type/role:** `telegram` — “Please type /start…”
- **Used when:** user has no session and didn’t send `/start`.
- **Edge cases:** user clicked inline buttons from old messages → may come via callback query; still gets chatId via fallback expression.

---

### Block 2.3 — State Router for Conversation Steps
**Overview:** Routes user input to the correct handler based on `current_step` stored in the session Data Table.  
**Nodes involved:** `Switch`

#### Switch
- **Type/role:** `switch` — state machine router.
- **Rules (in order):**
  1. `current_step == "awaiting_approval"` → `Check Approval Button`
  2. `current_step == "awaiting_name"` → `Upsert row(s)2` (save product name)
  3. `current_step == "awaiting_images"` → `If` (Done button check; else store image)
  4. `current_step == "awaiting_features"` → `If4` (None button check; else save features)
  5. `current_step == "awaiting_regular_price"` → `Update row(s)5` (save regular price)
  6. `current_step == "=awaiting_sale_price"` → `If5` (Same button check; else save sale price)
  7. `current_step == "complete"` → `Send a text message4`
- **Critical issue:** Rule 6 compares to `=awaiting_sale_price` (leading `=`). That likely prevents matching “awaiting_sale_price”, breaking the sale price step routing unless the table literally stores the `=` prefix (it does not; elsewhere it sets `awaiting_sale_price`).
- **Edge cases:** If `current_step` is empty/unknown, Switch routes nowhere (workflow ends silently).

---

### Block 2.4 — Step: Save Product Name → Ask for Images
**Overview:** Saves user-provided product name into session state and moves to image collection step.  
**Nodes involved:** `Upsert row(s)2`, `Send a text message1`

#### Upsert row(s)2
- **Type/role:** `dataTable` upsert — stores `product_name` and sets `current_step="awaiting_images"`.
- **product_name source:** `Telegram Trigger.message.text`
- **Edge cases:** if user sends `/start` again here, it will store `/start` as product name.

#### Send a text message1
- **Type/role:** `telegram` — instructs user to send product images and provides inline “Done” button.
- **replyMarkup:** inline keyboard with callback_data `"done"`.
- **Edge cases:** Telegram allows albums/media groups; bot will receive multiple messages or a media group event depending on client.

---

### Block 2.5 — Step: Collect Images (Store each upload) and Done Button
**Overview:** For each photo message, the workflow stores a selected file_id and its public file URL in a “User_Images” Data Table. When user clicks Done, it aggregates stored images into a comma-separated list and advances to features step.  
**Nodes involved:** `If`, `Code`, `Get a file`, `Edit Fields1`, `Insert row`, `Get row(s)3`, `Code in JavaScript3`, `Update row(s)1`, `Send a text message`

#### If (Done button check)
- **Type/role:** `if`
- **Condition:** callback_query.data == `"done"` (case-insensitive option enabled)
- **True path:** aggregate images (`Get row(s)3`).
- **False path:** process incoming photo (`Code`).

#### Code (select best photo size)
- **Type/role:** `code`
- **Logic:** From `Telegram Trigger.json.message.photo[]`, selects the largest `file_size` entry and stores `selected_file_id`.
- **Output:** to `Get a file`.
- **Edge cases:**
  - No `message.photo` (user sends text or document) → sets file_id null; later nodes may fail.
  - Media groups might arrive differently; relies on single message photo array.

#### Get a file
- **Type/role:** `telegram` resource=file — retrieves file_path for `selected_file_id`.
- **Input:** from `Code`
- **Failure modes:** invalid file_id, Telegram API failures.

#### Edit Fields1
- **Type/role:** `set` — builds a public Telegram file URL.
- **Value:** `https://api.telegram.org/file/bot{{your bot token}}/<file_path>`
- **Critical requirement:** must replace `{{your bot token}}` with a real token; otherwise URLs are unusable.
- **Output:** to `Insert row`.

#### Insert row
- **Type/role:** `dataTable` insert — writes one image row into “User_Images (Woocommerce)” table.
- **Columns:** `chat_id`, `file_id`, `image_url`
- **Edge cases:** duplicates are possible if user resends images; no dedupe.

#### Get row(s)3
- **Type/role:** `dataTable` get (returnAll = true) — loads all stored image rows for the chat_id.
- **Output:** to `Code in JavaScript3`

#### Code in JavaScript3 (aggregate)
- **Type/role:** `code`
- **Logic:** reduces all rows into:
  - `file_ids` = comma-separated file_id list
  - `image_urls` = comma-separated URL list
- **Output:** to `Update row(s)1`

#### Update row(s)1
- **Type/role:** `dataTable` update — stores aggregated `file_ids` and `image_urls` into the session table and sets `current_step="awaiting_features"`.
- **Output:** to `Send a text message` (features prompt)

#### Send a text message (features prompt)
- **Type/role:** `telegram`
- **Text:** asks for comma-separated features or inline “None” button (`callback_data="none"`).

---

### Block 2.6 — Step: Features (None vs Text) → Regular Price
**Overview:** Either records “no features” (via button) or saves typed features text; then asks for regular price and advances state.  
**Nodes involved:** `If4`, `Update row(s)3`, `Send a text message2`, `Update row(s)4`, `Send a text message6`

#### If4 (None button)
- **Type/role:** `if`
- **Condition:** callback_query.data == `"none"`
- **True path:** go straight to regular price step without features (`Update row(s)3`).
- **False path:** save typed features (`Update row(s)4`).

#### Update row(s)3
- **Type/role:** `dataTable` update — sets `current_step="awaiting_regular_price"`.
- **Output:** to `Send a text message2` (ask regular price)

#### Send a text message2
- **Type/role:** `telegram` — asks for regular price.

#### Update row(s)4
- **Type/role:** `dataTable` update — saves `features` from message text and sets `current_step="awaiting_regular_price"`.
- **Output:** to `Send a text message6` (also asks regular price; text is same as message2)

#### Send a text message6
- **Type/role:** `telegram` — asks for regular price (duplicate prompt node used for this branch).

---

### Block 2.7 — Step: Regular Price → Sale Price (“Same” or Text)
**Overview:** Saves regular price and asks for sale price, allowing a “Same” button to set sale price = regular price. Then advances to approval.  
**Nodes involved:** `Update row(s)5`, `Send a text message7`, `If5`, `Update row(s)6`, `Update row(s)7`, `Merge`, `Get All Data`, `Code in JavaScript4`, `If6`, `Telegram Group Media`, `Send a text message8`

#### Update row(s)5
- **Type/role:** `dataTable` update — saves `regular_price` from message text and sets `current_step="awaiting_sale_price"`.
- **Output:** to `Send a text message7`

#### Send a text message7
- **Type/role:** `telegram` — asks for sale price; provides “Same” inline button (`callback_data="same"`).

#### If5 (Same button)
- **Type/role:** `if`
- **Condition:** callback_query.data == `"same"` (case-insensitive)
- **True path:** `Update row(s)6` sets sale = regular.
- **False path:** `Update row(s)7` sets sale from message text and sets approval step.

#### Update row(s)6
- **Type/role:** `dataTable` update — sets `sale_price` equal to `Get User State1.json.regular_price`.
- **Issue:** Does **not** set `current_step="awaiting_approval"` here, unlike Update row(s)7. That means user might not advance to approval properly unless some other logic does it (it doesn’t).
- **Output:** to `Merge` input 0.

#### Update row(s)7
- **Type/role:** `dataTable` update — sets `sale_price` from message text and sets `current_step="awaiting_approval"`.
- **Output:** to `Merge` input 1.

#### Merge
- **Type/role:** `merge` — merges the two sale-price branches back together.
- **Config:** default merge behavior (in n8n, default is “Append” unless configured; JSON shows empty parameters).
- **Output:** to `Get All Data`
- **Edge cases:** if only one branch executes, merge behavior must support single input.

#### Get All Data
- **Type/role:** `dataTable` get (returnAll = true) — fetches all session rows (not filtered by chat_id).
- **Risk:** It pulls **all rows for all users** because there is no filter. The message preview can include another user’s data if multiple sessions exist.
- **Output:** to `Code in JavaScript4`

#### Code in JavaScript4 (split file_ids for Telegram preview)
- **Type/role:** `code`
- **Logic:** takes `file_ids` comma string → emits one item per id.
- **Output:** to `If6`

#### If6 (has images?)
- **Type/role:** `if`
- **Condition:** `$input.all().filter(valid id).length > 0`
- **True path:** `Telegram Group Media` (sends album preview)
- **No false path:** if no images, preview won’t send; approval step may not occur.

#### Telegram Group Media
- **Type/role:** `httpRequest` — calls Telegram `sendMediaGroup` directly.
- **Config:**
  - POST `https://api.telegram.org/bot[[bot token]]/sendMediaGroup`
  - Body builds `media` array from file_ids as `{type:"photo", media:"<file_id>"}`
- **Critical requirements:**
  - Replace `[[bot token]]` with real token.
  - Telegram `sendMediaGroup` expects file_id or URL; file_id works.
- **Edge cases:** Telegram limits media group size (max 10). This workflow can store many images, but later code caps to 10 URLs in some places; still preview may exceed 10 if more file_ids are saved.

#### Send a text message8 (review + approval buttons)
- **Type/role:** `telegram`
- **Text:** renders product_name, features, regular_price, sale_price from `Get All Data` (again, risky without filtering).
- **Buttons:** Approve (`approve_product`) and Cancel & Restart (`cancel_product`)
- **executeOnce:** true (sends only once per execution)
- **Edge cases:** Cancel path is implemented later but only checks approval; cancel handling is incomplete (see Block 2.8).

---

### Block 2.8 — Approval / Cancel Routing
**Overview:** Processes approval callback and starts product creation, or cancels and resets to first step.  
**Nodes involved:** `Check Approval Button`, `Send a text message9`, `Update row(s)`, `Get row(s)`, `Code in JavaScript`, `Edit Fields`, `Code in JavaScript2`, `HTTP Request`, `If2`, (and downstream AI blocks), plus cancel branch `Upsert row(s)1`, `Send a text message10`

#### Check Approval Button
- **Type/role:** `if`
- **Condition:** callback_query.data == `"approve_product"`
- **True path:** proceed with product creation (`Send a text message9`)
- **False path:** `Upsert row(s)1` (intended cancel/restart)
- **Gap:** It never checks for `cancel_product` specifically; *any* non-approve callback leads to restart logic.

#### Send a text message9
- **Type/role:** `telegram`
- **Text:** “Creating your product now...”
- **Output:** to `Update row(s)` (sets session complete)

#### Update row(s)
- **Type/role:** `dataTable` update — sets `current_step="complete"`.
- **Output:** to `Get row(s)` (fetch session row by chat_id)

#### Get row(s)
- **Type/role:** `dataTable` get — fetches session row by chat_id (filtered).
- **Output:** to `Code in JavaScript`

#### Code in JavaScript (image_urls → image_url1..N object)
- **Type/role:** `code`
- **Logic:** takes `image_urls` comma string → returns `{image_url1:..., image_url2:...}` etc.
- **Output:** to `Edit Fields`
- **Edge cases:** empty image_urls returns `{}`.

#### Edit Fields (compose all core fields + image_url1..10 passthrough)
- **Type/role:** `set`
- **Produces fields:**
  - `chatId`, `features`, `title`, `regular_price`, `sale price` (note space), `image1_url..image10_url`
- **Key dependency issues:**
  - References `$('Code1').item.json.chat_id` but `Code1` is part of the earlier state-loading branch; in approval path, this may not be in the execution data chain.
  - Uses `$('Get User State').item.json.product_name` for title (also earlier branch).
  - Uses `$('Code1').item.json.regular_price` and `.sale_price` (likely wrong; Code1 returns user state, not necessarily those fields).
  - The field name `"sale price"` contains a space; downstream expects `sale_price` in WooCommerce requests, which uses `$json.sale_price` from table rows, not this set field.
- **Output:** to `Code in JavaScript2`
- **Risk:** This node mixes data sources inconsistently; however downstream product creation ultimately pulls from session table again.

#### Code in JavaScript2 (build WooCommerce images array)
- **Type/role:** `code`
- **Logic:** scans `image1_url..image10_url` and creates `images=[{src:url}, ...]`.
- **Output:** to `HTTP Request`
- **Note:** This images array is not actually used in the WooCommerce requests in this workflow; later WooCommerce request nodes use `Code in JavaScript6/7` outputs instead.

#### HTTP Request (download image1)
- **Type/role:** `httpRequest`
- **URL:** `$json.image1_url`
- **Response:** file (binary)
- **Purpose:** provides an image input for “no features” description-generation path.
- **Edge cases:** image1_url missing/invalid → node fails.

#### If2 (features provided?)
- **Type/role:** `if`
- **Condition:** `Edit Fields.json.features` is not empty
- **True path:** “features provided” AI copywriting (`Product Description Generation`)
- **False path:** “no features” image-based copywriting (`Product Description Generation1`)

---

### Block 2.9 — AI Copywriting (Features Provided Branch)
**Overview:** Uses Gemini to generate long description, short description, and slug from product title + features. Also generates per-image SEO title/alt tags and creates the WooCommerce product.  
**Nodes involved:** `Product Description Generation`, `Google Gemini Chat Model`, `Short Description Generation`, `Google Gemini Chat Model1`, `Slug Generation`, `Google Gemini Chat Model3`, `Get row(s)5`, `Code in JavaScript8`, `HTTP Request5`, `AI Agent1`, `Google Gemini Chat Model8`, `Structured Output Parser1`, `Code in JavaScript7`, `Get row(s)4`, `HTTP Request4`, `Send a text message13`

#### Product Description Generation
- **Type/role:** `chainLlm` — prompt-defined content generation.
- **Inputs used:** `Edit Fields.title`, `Edit Fields.features`
- **Model:** `Google Gemini Chat Model`
- **Output:** `.json.text` used later.
- **Edge cases:** prompt forbids inventing features; if features are vague, output quality degrades.

#### Short Description Generation
- **Type/role:** `chainLlm` — short description paragraph.
- **Model:** `Google Gemini Chat Model1`

#### Slug Generation
- **Type/role:** `chainLlm` — creates SEO slug.
- **Model:** `Google Gemini Chat Model3`
- **Output connection:** to `Get row(s)5` (note: this is “features branch” but connects to `Get row(s)5`, which is also used in the other branch; naming is confusing.)

#### Get row(s)5
- **Type/role:** `dataTable` get by chat_id — obtains session row including `image_urls` and `product_name`.
- **Output:** to `Code in JavaScript8`

#### Code in JavaScript8 (split image_urls into items)
- **Type/role:** `code`
- **Logic:** outputs one item per `image_url`.
- **Output:** to `HTTP Request5`

#### HTTP Request5 (fetch image URL content)
- **Type/role:** `httpRequest`
- **URL:** `{{$json.image_url}}`
- **Output:** to `AI Agent1`
- **Note:** Not configured to return binary explicitly; it may return HTML/bytes depending on server. AI Agent prompt references `$json.data || $json.image_url || $json`, so it might just pass URL.

#### AI Agent1 (image title/alt generation)
- **Type/role:** `langchain.agent`
- **Model:** `Google Gemini Chat Model8`
- **Output parser:** `Structured Output Parser1` expecting JSON with `{title, alt_tag}`.
- **Prompt:** uses product_name and “image provided here …”.
- **Output:** goes to `Code in JavaScript7` to merge with URLs.

#### Structured Output Parser1
- **Type/role:** structured JSON parsing for AI Agent1.
- **Schema example:** `{"title":"","alt_tag":""}`
- **Edge cases:** model returns invalid JSON → parser errors.

#### Code in JavaScript7 (merge URL + AI metadata)
- **Type/role:** `code`
- **Logic:** merges image_url items from `HTTP Request5` with agent outputs, producing:
  - `images: [{src, name, alt}, ...]`
- **Output:** to `Get row(s)4`

#### Get row(s)4
- **Type/role:** `dataTable` get (returnAll=true) — **unfiltered** fetch of all sessions.
- **executeOnce:** true
- **Risk:** same as `Get All Data`—can pick wrong user row.
- **Output:** to `HTTP Request4`

#### HTTP Request4 (WooCommerce create product, features branch)
- **Type/role:** `httpRequest` POST with WooCommerce predefined credentials.
- **Endpoint:** `https://yourwp-website/wp-json/wc/v3/products` (placeholder must be replaced).
- **Body parameters:**
  - `name = $json.product_name` (from `Get row(s)4`)
  - `status = draft`
  - `description = Product Description Generation.text`
  - `short_description = Short Description Generation.text`
  - `slug = Slug Generation.text`
  - `images = Code in JavaScript7.images`
  - `regular_price`, `sale_price`
- **Failure modes:**
  - Wrong base URL, invalid WooCommerce API keys, SSL issues.
  - WooCommerce rejects invalid prices (must be string numeric), invalid images array, or duplicate slug.

#### Send a text message13
- **Type/role:** `telegram` — confirms product created (draft).
- **Uses:** `$json.name` returned by WooCommerce.

---

### Block 2.10 — AI Copywriting (No Features Branch / Image-Based)
**Overview:** When features are not provided, the workflow downloads the first image and uses it to generate descriptions; it still generates slug from title, generates per-image title/alt, then creates the WooCommerce product.  
**Nodes involved:** `Product Description Generation1`, `Google Gemini Chat Model2`, `Short Description Generation1`, `Google Gemini Chat Model4`, `Slug Generation1`, `Google Gemini Chat Model5`, `Get row(s)2`, `Code in JavaScript5`, `HTTP Request2`, `AI Agent`, `Google Gemini Chat Model7`, `Structured Output Parser`, `Code in JavaScript6`, `Get row(s)1`, `HTTP Request3`, `Send a text message12`

#### Product Description Generation1
- **Type/role:** `chainLlm` — image-based long description.
- **Input references:** `HTTP Request.binary.data` + title.
- **Model:** `Google Gemini Chat Model2`
- **Edge cases:** If Gemini node does not support binary image input in this configuration, output may be poor or fail; depends on node capabilities/version.

#### Short Description Generation1
- **Type/role:** `chainLlm` — image-based short description.
- **Model:** `Google Gemini Chat Model4`

#### Slug Generation1
- **Type/role:** `chainLlm` — slug from title only.
- **Model:** `Google Gemini Chat Model5`

#### Get row(s)2 → Code in JavaScript5 → HTTP Request2 → AI Agent → Structured Output Parser → Code in JavaScript6
- **Role:** Same as features branch, but using the “2” numbered nodes:
  - Split image_urls into items
  - Fetch each URL (HTTP Request2)
  - AI Agent generates title/alt (with output parser)
  - Code merges into `images` array
- **Key node details:**
  - **AI Agent** uses `Get row(s)2.item.json.product_name` in prompt.
  - **Structured Output Parser** expects `{title, alt_tag}`.
  - **Code in JavaScript6** merges `HTTP Request2` items and parsed outputs into `{images:[{src,name,alt}]}`.

#### Get row(s)1
- **Type/role:** `dataTable` get (returnAll=true), executeOnce true — **unfiltered** sessions (risk).
- **Output:** to `HTTP Request3`

#### HTTP Request3 (WooCommerce create product, no-features branch)
- **Type/role:** `httpRequest` POST to WooCommerce.
- **Endpoint:** `https://your-wp-site/wp-json/wc/v3/products` (placeholder).
- **Body parameters:** similar to features branch but uses:
  - `description = Product Description Generation1.text`
  - `short_description = Short Description Generation1.text`
  - `slug = Slug Generation1.text`
  - `images = Code in JavaScript6.images`
- **Output:** to `Send a text message12`

#### Send a text message12
- **Type/role:** `telegram` — confirms product created (draft).

---

### Block 2.11 — Completion Message + Cleanup
**Overview:** After setting state to `complete`, user is told they can start again; then session and images tables are cleared.  
**Nodes involved:** `Send a text message4`, `Delete row(s)`, `Delete row(s)1`

#### Send a text message4
- **Type/role:** `telegram` — “previous product created… /start…”
- **Triggered by:** Switch rule `current_step=="complete"`

#### Delete row(s)
- **Type/role:** `dataTable` deleteRows — deletes session row for this chat_id from “WooCommerce Product Manager”
- **Output:** to `Delete row(s)1`

#### Delete row(s)1
- **Type/role:** `dataTable` deleteRows — deletes image rows from “User_Images (Woocommerce)”

**Note:** In the JSON connections, `Send a text message4` leads to `Delete row(s)` (cleanup). However, because of earlier issues (state advancement on sale “Same”, switch rule typo), `complete` may not be reached reliably.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point (messages + callbacks) | — | Get User State | ## Bot Trigger & State Query<br>Telegram trigger polls for callback queries and messages and checks the current status of user in the process of product creation. |
| Get User State | dataTable (get) | Load session by chat_id | Telegram Trigger | If /abort | ## Bot Trigger & State Query<br>Telegram trigger polls for callback queries and messages and checks the current status of user in the process of product creation. |
| If /abort | if | Abort command detection | Get User State | Delete row(s)2; Code1 | ## Check for Abort Command<br>If during the process of product creation user uses abort command, then it resets both data tables and notifies user about it |
| Delete row(s)2 | dataTable (deleteRows) | Delete session state | If /abort | Delete row(s)3 | ## Check for Abort Command<br>If during the process of product creation user uses abort command, then it resets both data tables and notifies user about it |
| Delete row(s)3 | dataTable (deleteRows) | Delete temporary images | Delete row(s)2 | Send a text message11 | ## Check for Abort Command<br>If during the process of product creation user uses abort command, then it resets both data tables and notifies user about it |
| Send a text message11 | telegram | Notify abort | Delete row(s)3 | — | ## Check for Abort Command<br>If during the process of product creation user uses abort command, then it resets both data tables and notifies user about it |
| Code1 | code | Pass through session items | If /abort (false) | If3 |  |
| If3 | if | Check session exists | Code1 | Get User State1; If1 | ## Check for Chat ID field in WooCommerce Product Manager Data Table<br>If a user is in the middle of product creation process then it is passed through to check for relevant steps in product creation process, if not then it's diverted user to prompt for a fresh start. |
| Get User State1 | dataTable (get, returnAll) | Reload session incl. current_step | If3 (true) | Switch | ## Check for Chat ID field in WooCommerce Product Manager Data Table<br>If a user is in the middle of product creation process then it is passed through to check for relevant steps in product creation process, if not then it's diverted user to prompt for a fresh start. |
| If1 | if | Check /start command | If3 (false) | Send a text message3; Send a text message5 | ## Check for /start in Telegram Message<br>If /start is found… step is Update to awaiting name… |
| Send a text message3 | telegram | Ask for product name | If1 (true) | Upsert row(s) | ## Check for /start in Telegram Message<br>If /start is found… step is Update to awaiting name… |
| Upsert row(s) | dataTable (upsert) | Initialize session (awaiting_name) | Send a text message3 | — | ## Check for /start in Telegram Message<br>If /start is found… step is Update to awaiting name… |
| Send a text message5 | telegram | Prompt user to /start | If1 (false) | — | ## Check for /start in Telegram Message<brIf /start is not found… user is prompted to use the start command. |
| Switch | switch | Route by current_step | Get User State1 | Check Approval Button; Upsert row(s)2; If; If4; Update row(s)5; If5; Send a text message4 |  |
| Upsert row(s)2 | dataTable (upsert) | Save product name, step→awaiting_images | Switch | Send a text message1 | ## Asks for Product Name<br>Prompts the user to enter product name and changes the current step in data table to awaiting images |
| Send a text message1 | telegram | Ask user to upload images + Done button | Upsert row(s)2 | — |  |
| If | if | Detect Done button vs image upload | Switch | Get row(s)3; Code |  |
| Code | code | Choose largest photo file_id | If (false) | Get a file | ## If User hasn't clicked Done button after Uploading Images on Telegram bot<br>Image URLs are added into the user images data table for each image |
| Get a file | telegram (file) | Fetch Telegram file_path | Code | Edit Fields1 | ## If User hasn't clicked Done button after Uploading Images on Telegram bot<br>Image URLs are added into the user images data table for each image |
| Edit Fields1 | set | Build Telegram file URL | Get a file | Insert row | ## If User hasn't clicked Done button after Uploading Images on Telegram bot<br>Image URLs are added into the user images data table for each image |
| Insert row | dataTable (insert) | Store one image row | Edit Fields1 | — | ## If User hasn't clicked Done button after Uploading Images on Telegram bot<br>Image URLs are added into the user images data table for each image |
| Get row(s)3 | dataTable (get, returnAll) | Load all stored images for chat | If (true) | Code in JavaScript3 | ## If User has clicked Done button after Uploading Images on Telegram bot<br>Image URLs… separated by a comma… |
| Code in JavaScript3 | code | Aggregate file_ids/image_urls strings | Get row(s)3 | Update row(s)1 | ## If User has clicked Done button after Uploading Images on Telegram bot<br>Image URLs… separated by a comma… |
| Update row(s)1 | dataTable (update) | Save image_urls & step→awaiting_features | Code in JavaScript3 | Send a text message | ## If User has clicked Done button after Uploading Images on Telegram bot<br>…user is prompted to go to next step |
| Send a text message | telegram | Ask features or None button | Update row(s)1 | — |  |
| If4 | if | Detect “None” features | Switch | Update row(s)3; Update row(s)4 |  |
| Update row(s)3 | dataTable (update) | Step→awaiting_regular_price (no features) | If4 (true) | Send a text message2 | ## If User has clicked None Button for Feature Availability… |
| Send a text message2 | telegram | Ask regular price | Update row(s)3 | — |  |
| Update row(s)4 | dataTable (update) | Save features + step→awaiting_regular_price | If4 (false) | Send a text message6 | ## If User has Provided Features… |
| Send a text message6 | telegram | Ask regular price | Update row(s)4 | — |  |
| Update row(s)5 | dataTable (update) | Save regular_price + step→awaiting_sale_price | Switch | Send a text message7 | ## User Provides Regular Price… |
| Send a text message7 | telegram | Ask sale price + Same button | Update row(s)5 | — |  |
| If5 | if | Detect “Same” sale price | Switch | Update row(s)6; Update row(s)7 | ## Checks if user presses "Same" button for Sale Price |
| Update row(s)6 | dataTable (update) | sale_price = regular_price | If5 (true) | Merge | ## User Provides Sale Price… |
| Update row(s)7 | dataTable (update) | Save sale_price + step→awaiting_approval | If5 (false) | Merge | ## User Provides Sale Price… |
| Merge | merge | Join sale price branches | Update row(s)6; Update row(s)7 | Get All Data |  |
| Get All Data | dataTable (get, returnAll) | Fetch sessions (unfiltered) | Merge | Code in JavaScript4 |  |
| Code in JavaScript4 | code | Split file_ids to items for preview | Get All Data | If6 |  |
| If6 | if | Check at least 1 file_id | Code in JavaScript4 | Telegram Group Media |  |
| Telegram Group Media | httpRequest | Send media group preview | If6 | Send a text message8 |  |
| Send a text message8 | telegram | Review summary + Approve/Cancel buttons | Telegram Group Media | — | ## User Provides Sale Price… |
| Check Approval Button | if | Detect approve_product | Switch | Send a text message9; Upsert row(s)1 | # Check for approve_product in Telegram Callback Query<br>If Appove button is pressed… |
| Send a text message9 | telegram | Notify product creation started | Check Approval Button (true) | Update row(s) | # Check for approve_product… |
| Upsert row(s)1 | dataTable (upsert) | Reset to awaiting_name (cancel path) | Check Approval Button (false) | Send a text message10 | ## If Disapproved… |
| Send a text message10 | telegram | Cancelled, ask for product name | Upsert row(s)1 | — | ## If Disapproved… |
| Update row(s) | dataTable (update) | Step→complete | Send a text message9 | Get row(s) |  |
| Get row(s) | dataTable (get) | Fetch session by chat_id | Update row(s) | Code in JavaScript |  |
| Code in JavaScript | code | Convert image_urls to image_url1..N | Get row(s) | Edit Fields |  |
| Edit Fields | set | Compose fields for AI/copy steps | Code in JavaScript | Code in JavaScript2 |  |
| Code in JavaScript2 | code | Build images=[{src}] from image1..10 | Edit Fields | HTTP Request |  |
| HTTP Request | httpRequest (file) | Download image1 for image-based copy | Code in JavaScript2 | If2 |  |
| If2 | if | Branch on features presence | HTTP Request | Product Description Generation; Product Description Generation1 | ## Check for Features<brChecks if features are provided or not by the user |
| Product Description Generation | chainLlm | Long description from title+features | If2 (true) | Short Description Generation | ## If Features are provided… |
| Short Description Generation | chainLlm | Short description from title+features | Product Description Generation | Slug Generation | ## If Features are provided… |
| Slug Generation | chainLlm | Slug from title+features | Short Description Generation | Get row(s)5 | ## If Features are provided… |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM for Product Description Generation | — | (ai_languageModel) Product Description Generation | ## If Features are provided… |
| Google Gemini Chat Model1 | lmChatGoogleGemini | LLM for Short Description Generation | — | (ai_languageModel) Short Description Generation | ## If Features are provided… |
| Google Gemini Chat Model3 | lmChatGoogleGemini | LLM for Slug Generation | — | (ai_languageModel) Slug Generation | ## If Features are provided… |
| Get row(s)5 | dataTable (get) | Session row for image_urls (features branch) | Slug Generation | Code in JavaScript8 | ## If Features are provided… |
| Code in JavaScript8 | code | Split image_urls → items | Get row(s)5 | HTTP Request5 | ## If Features are provided… |
| HTTP Request5 | httpRequest | Fetch image URL data | Code in JavaScript8 | AI Agent1 | ## If Features are provided… |
| AI Agent1 | langchain.agent | Generate per-image title/alt (features branch) | HTTP Request5 | Code in JavaScript7 | ## If Features are provided… |
| Google Gemini Chat Model8 | lmChatGoogleGemini | LLM for AI Agent1 | — | (ai_languageModel) AI Agent1 | ## If Features are provided… |
| Structured Output Parser1 | outputParserStructured | Enforce JSON {title,alt_tag} | — | (ai_outputParser) AI Agent1 | ## If Features are provided… |
| Code in JavaScript7 | code | Merge URLs + AI metadata → images array | AI Agent1 | Get row(s)4 | ## If Features are provided… |
| Get row(s)4 | dataTable (get, returnAll) | Unfiltered sessions (risk) | Code in JavaScript7 | HTTP Request4 | ## If Features are provided… |
| HTTP Request4 | httpRequest | WooCommerce create product (features) | Get row(s)4 | Send a text message13 | ## If Features are provided… |
| Send a text message13 | telegram | Notify WooCommerce draft created | HTTP Request4 | — | ## If Features are provided… |
| Product Description Generation1 | chainLlm | Long description from image + title | If2 (false) | Short Description Generation1 | ## If Features are not provided… |
| Short Description Generation1 | chainLlm | Short description from image + title | Product Description Generation1 | Slug Generation1 | ## If Features are not provided… |
| Slug Generation1 | chainLlm | Slug from title | Short Description Generation1 | Get row(s)2 | ## If Features are not provided… |
| Google Gemini Chat Model2 | lmChatGoogleGemini | LLM for Product Description Generation1 | — | (ai_languageModel) Product Description Generation1 | ## If Features are not provided… |
| Google Gemini Chat Model4 | lmChatGoogleGemini | LLM for Short Description Generation1 | — | (ai_languageModel) Short Description Generation1 | ## If Features are not provided… |
| Google Gemini Chat Model5 | lmChatGoogleGemini | LLM for Slug Generation1 | — | (ai_languageModel) Slug Generation1 | ## If Features are not provided… |
| Get row(s)2 | dataTable (get) | Session row for image_urls (no-features branch) | Slug Generation1 | Code in JavaScript5 | ## If Features are not provided… |
| Code in JavaScript5 | code | Split image_urls → items | Get row(s)2 | HTTP Request2 | ## If Features are not provided… |
| HTTP Request2 | httpRequest | Fetch image URL data | Code in JavaScript5 | AI Agent | ## If Features are not provided… |
| AI Agent | langchain.agent | Generate per-image title/alt (no-features) | HTTP Request2 | Code in JavaScript6 | ## If Features are not provided… |
| Google Gemini Chat Model7 | lmChatGoogleGemini | LLM for AI Agent | — | (ai_languageModel) AI Agent | ## If Features are not provided… |
| Structured Output Parser | outputParserStructured | Enforce JSON {title,alt_tag} | — | (ai_outputParser) AI Agent | ## If Features are not provided… |
| Code in JavaScript6 | code | Merge URLs + AI metadata → images array | AI Agent | Get row(s)1 | ## If Features are not provided… |
| Get row(s)1 | dataTable (get, returnAll) | Unfiltered sessions (risk) | Code in JavaScript6 | HTTP Request3 | ## If Features are not provided… |
| HTTP Request3 | httpRequest | WooCommerce create product (no-features) | Get row(s)1 | Send a text message12 | ## If Features are not provided… |
| Send a text message12 | telegram | Notify WooCommerce draft created | HTTP Request3 | — | ## If Features are not provided… |
| Send a text message4 | telegram | Completion prompt | Switch | Delete row(s) | ## Prompts User about completion and Resets Data Table… |
| Delete row(s) | dataTable (deleteRows) | Cleanup session | Send a text message4 | Delete row(s)1 | ## Prompts User about completion and Resets Data Table… |
| Delete row(s)1 | dataTable (deleteRows) | Cleanup user images | Delete row(s) | — | ## Prompts User about completion and Resets Data Table… |
| Sticky Note | stickyNote | Documentation note | — | — | # Telegram Bot based WooCommerce Product Creator… |
| Sticky Note1..18 | stickyNote | Section comments | — | — | (content captured above per affected nodes) |

---

## 4. Reproducing the Workflow from Scratch (Step-by-Step)

1. **Create Telegram Bot**
   1. Use **@BotFather** to create a bot and get the token.
   2. In n8n, create **Telegram credentials** (token-based).

2. **Create Data Tables in n8n**
   1. Data Table A: **WooCommerce Product Manager**
      - Fields (String unless noted): `chat_id`, `current_step`, `product_name`, `image_urls`, `features`, `regular_price`, `sale_price`, `file_ids`
   2. Data Table B: **User_Images (Woocommerce)**
      - Fields: `chat_id`, `file_id`, `image_url`

3. **Create WooCommerce API Credentials**
   1. In WooCommerce: create REST API keys with **Read/Write**.
   2. In n8n: create **WooCommerce API credential** (predefined credential type).

4. **Telegram Trigger**
   1. Add **Telegram Trigger** node.
   2. Updates: enable `message` and `callback_query`.
   3. Select Telegram credentials.

5. **Load Session**
   1. Add Data Table node **Get User State** (operation: get).
   2. Filter: `chat_id = {{ chat.id from message or callback_query.message }}`.
   3. Enable **Always Output Data**.

6. **Abort Handling**
   1. Add **If /abort** node:
      - Condition: if message.text or callback_query.data equals `/abort`.
   2. True branch:
      - Data Table **Delete row(s)2**: delete from Product Manager where chat_id matches.
      - Data Table **Delete row(s)3**: delete from User_Images where chat_id matches.
      - Telegram **Send a text message11**: “aborted…” (ensure you set `chatId`).
   3. False branch continues to normal routing.

7. **Session Exists Check**
   1. Add **Code1** node: return `$('Get User State').all()`.
   2. Add **If3** node: condition `Get User State.chat_id not empty`.
   3. True → Data Table **Get User State1** (get returnAll=true, filter by chat_id) → to Switch.
   4. False → **If1** for /start gating.

8. **/start Gate**
   1. **If1**: message.text equals `/start`.
   2. True:
      - Telegram **Send a text message3** asking for product name.
      - Data Table **Upsert row(s)** set `chat_id` and `current_step="awaiting_name"`.
   3. False:
      - Telegram **Send a text message5** asking user to type /start.

9. **Switch State Machine**
   1. Add **Switch**:
      - Use expressions reading `Get User State1.json.current_step`.
      - Add rules for: `awaiting_name`, `awaiting_images`, `awaiting_features`, `awaiting_regular_price`, `awaiting_sale_price`, `awaiting_approval`, `complete`.
   2. Ensure the sale price rule is exactly `"awaiting_sale_price"` (no leading `=`).

10. **Product Name Step**
   1. Data Table **Upsert row(s)2**:
      - Save `product_name = Telegram message.text`
      - Set `current_step="awaiting_images"`
   2. Telegram **Send a text message1**:
      - Prompt for images
      - Inline keyboard button “Done” with callback_data `done`

11. **Image Collection Step**
   1. **If** node:
      - If callback_query.data == `done` → aggregation branch
      - Else → per-photo storage branch
   2. Per-photo storage:
      - **Code** node: select largest photo size file_id.
      - Telegram **Get a file** (resource=file) using selected file_id.
      - **Set** node “Edit Fields1” to build `new_image_url`:
        - Use `https://api.telegram.org/file/bot<YOUR_TOKEN>/{{$json.result.file_path}}`
      - Data Table **Insert row** into User_Images table (chat_id, file_id, image_url).
   3. Done branch:
      - Data Table **Get row(s)3** from User_Images by chat_id (returnAll=true).
      - **Code in JavaScript3** aggregate into comma strings (`file_ids`, `image_urls`).
      - Data Table **Update row(s)1** in Product Manager: store file_ids, image_urls; set `current_step="awaiting_features"`.
      - Telegram **Send a text message**: ask for features with “None” button (callback_data `none`).

12. **Features Step**
   1. **If4**: callback_query.data == `none`
   2. True:
      - Data Table **Update row(s)3**: set `current_step="awaiting_regular_price"`
      - Telegram **Send a text message2**: ask regular price
   3. False:
      - Data Table **Update row(s)4**: set `features = message.text`, `current_step="awaiting_regular_price"`
      - Telegram **Send a text message6**: ask regular price

13. **Regular Price Step**
   1. Data Table **Update row(s)5**: set regular_price from message.text; set `current_step="awaiting_sale_price"`
   2. Telegram **Send a text message7** with “Same” button (callback_data `same`).

14. **Sale Price Step**
   1. **If5**: callback_query.data == `same`
   2. True:
      - Data Table **Update row(s)6**: set sale_price = regular_price; also set `current_step="awaiting_approval"` (needed fix).
   3. False:
      - Data Table **Update row(s)7**: set sale_price = message.text; set `current_step="awaiting_approval"`
   4. Merge the branches with **Merge** node.
   5. Fetch session data filtered by chat_id (recommended fix; avoid unfiltered “Get All Data”).
   6. Build Telegram media group preview:
      - Split file_ids to items (Code node)
      - HTTP Request to Telegram sendMediaGroup with bot token
   7. Send review message with Approve/Cancel buttons.

15. **Approval Handling**
   1. **Check Approval Button**:
      - If approve_product: proceed
      - If cancel_product: reset
   2. Approve path:
      - Telegram “Creating…” message
      - Set session complete (or move to a “creating” step)
      - Start AI + WooCommerce creation (below)
   3. Cancel path:
      - Upsert session `current_step="awaiting_name"`
      - Telegram “Action cancelled… product name?”

16. **AI + WooCommerce Creation**
   - **Branch A (features provided):**
     1. Chain LLM: Product Description Generation (Gemini)
     2. Chain LLM: Short Description Generation (Gemini)
     3. Chain LLM: Slug Generation (Gemini)
   - **Branch B (no features):**
     1. HTTP Request download image1 as binary
     2. Chain LLM: Product Description Generation1 (Gemini, image-based)
     3. Chain LLM: Short Description Generation1
     4. Chain LLM: Slug Generation1
   - **Images SEO (both branches):**
     1. Split image_urls into items
     2. HTTP Request per image URL (optional)
     3. AI Agent (Gemini) with **Structured Output Parser** requiring `{title, alt_tag}`
     4. Code node to merge URL + title/alt into `images` array objects `{src,name,alt}`
   - **WooCommerce create product:**
     1. HTTP Request POST to `https://<your-site>/wp-json/wc/v3/products`
     2. Auth: predefined WooCommerce credentials
     3. JSON body includes `name`, `status:"draft"`, `description`, `short_description`, `slug`, `images`, `regular_price`, `sale_price`
   - **Telegram confirmation message**.

17. **Cleanup**
   1. Delete session row from Product Manager table.
   2. Delete rows from User_Images table.
   3. Prompt user to `/start` again.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer (provided by user) |
| Ping me on X | https://x.com/matta_kshitij |
| Refer to n8n community | https://community.n8n.io/ |
| Replace placeholder URLs and bot tokens | Sticky note: initial setup requires replacing `https://your-wp-site/` and Telegram token placeholders |

