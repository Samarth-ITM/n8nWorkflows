Find AliExpress affiliate products via Telegram with OpenAI and Decodo

https://n8nworkflows.xyz/workflows/find-aliexpress-affiliate-products-via-telegram-with-openai-and-decodo-12437


# Find AliExpress affiliate products via Telegram with OpenAI and Decodo

## 1. Workflow Overview

**Purpose:**  
This workflow turns a Telegram group into an AliExpress affiliate product finder. Users send a Hebrew “starter phrase” message (e.g., “תמצא לי …”), the workflow moderates the content with OpenAI, verifies the user is a member of a required Telegram channel/group, scrapes AliExpress search results via Decodo, generates AliExpress affiliate links, and posts product “cards” (photo + persuasive Hebrew caption + buttons). Users can request more results via an inline button (callback query).

**Target use cases:**
- Affiliate marketing in a Telegram community (product discovery + tracked links).
- Moderated “shopping assistant” bot that blocks links/spam/inappropriate requests.
- Enforcing “must join channel first” gating before allowing bot usage.

### 1.1 Logical Blocks
1. **Input Reception & Basic Filtering**: Receive Telegram messages/callbacks; ignore `/start`; split message vs callback flows.
2. **AI Content Moderation**: OpenAI classifies message as approved/rejected (links and disallowed content rejected).
3. **Membership Gating**: Check whether user is a member of a specific Telegram channel/group; if not, prompt to join.
4. **User Request Validation (Starter Phrase)**: Only proceed if message begins with allowed Hebrew starters; otherwise ask for correct wording.
5. **Search Query Cleanup (OpenAI)**: Extract and clean product query text for AliExpress search.
6. **AliExpress Scraping (Decodo) + Product Extraction (Code)**: Scrape AliExpress HTML and parse embedded JSON for products.
7. **Selection Logic (“Top 2” vs “More results”)**:  
   - Initial message: take first 2 products.  
   - “More results” callback: pick 2 random items from positions 3–7.
8. **Affiliate Link Generation + Error Handling**: Generate tracking links; if generation fails for all items, fall back to re-extract/retry path.
9. **Message Composition & Response Delivery**: OpenAI writes a persuasive Hebrew caption; Telegram sends photo message with inline buttons; temporary “please wait” messages are deleted.
10. **Enforcement Action (Auto-remove)**: If moderation rejects content, warn/delete and ban user via Telegram HTTP API.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception & Basic Filtering
**Overview:** Receives updates from Telegram (messages and inline button clicks). Filters out `/start` and routes either the “message” or “callback query” path.

**Nodes involved:**
- Telegram Trigger
- Bot start filtering
- If

#### Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point webhook for Telegram updates.
- **Config:** Listens to `message` and `callback_query`.
- **Outputs:** Sends a single item containing either `message` or `callback_query`.
- **Edge cases:** Telegram credential misconfig; webhook not registered; bot not added to group or missing permissions.

#### Bot start filtering
- **Type / role:** `filter` — blocks `/start`.
- **Config:** Condition `$json.message.text != "/start"`.
- **Input:** Telegram Trigger (message updates).
- **Output:** Passes non-start messages onward.
- **Edge cases:** If update is a callback query, `$json.message` may not exist (depends on n8n trigger output shape). In this workflow, callback flow is still routed after `If`, but the filter node assumes `message.text` exists.

#### If
- **Type / role:** `if` — routes message vs callback.
- **Config choice:** Checks **callback_query.data does not exist** (`notExists`) to identify a normal message.
- **Outputs:**
  - **True branch:** message flow → “Message a model” (moderation).
  - **False branch:** callback flow → “Only if the person who asked for more…”.
- **Edge cases:** Telegram callback payload differences; expression failures if `callback_query` path is missing but `notExists` is evaluated incorrectly.

---

### Block 2.2 — AI Content Moderation (Message Flow)
**Overview:** Uses OpenAI to reject messages containing links or disallowed content. Approved messages continue to membership check; rejected messages trigger removal actions.

**Nodes involved:**
- Message a model
- Code in JavaScript1
- If there is problematic content
- Opening message 1
- Deleting a message 2
- Automatic removal of the user from the group

#### Message a model
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — moderation classification.
- **Config:**
  - Model: `gpt-5.2`.
  - System prompt enforces Hebrew-only JSON output with `{status: approved|rejected}`; automatic rejection for any URL.
  - User content: `{{$json.message.text}}`.
- **Output:** LangChain/OpenAI structured output in `output[0].content[0].text` (string).
- **Edge cases:** Model may return non-JSON or Markdown-wrapped JSON; API timeouts; cost/rate limits.

#### Code in JavaScript1
- **Type / role:** `code` — parses the model output into real JSON fields.
- **Config choices:**
  - Reads `rawText = $input.first().json.output[0].content[0].text`.
  - Strips ```json fences and `JSON.parse`.
- **Output:** Returns parsed object (e.g., `{status, message, reason}`).
- **Failure types:** `JSON.parse` throws if model returns invalid JSON; missing output path.

#### If there is problematic content
- **Type / role:** `if` — routes approved vs rejected.
- **Config:** Checks `$json.status != "rejected"` (so **true = approved**).
- **Outputs:**
  - **True:** go to membership check (“Checking if we are in a certain group”).
  - **False:** send rejection message and enforce removal.

#### Opening message 1
- **Type / role:** `telegram` — sends the rejection message returned by moderation.
- **Config:** `text = {{$json.message}}`, to the original `message.chat.id`.
- **Edge cases:** If moderation returns `message: null` for approved, but this node is only hit on rejected branch.

#### Deleting a message 2
- **Type / role:** `telegram` deleteMessage — deletes the offending user message.
- **Config:** `messageId = Telegram Trigger.message.message_id`.
- **Requires:** Bot must have admin rights to delete messages in group/supergroup.

#### Automatic removal of the user from the group
- **Type / role:** `httpRequest` — calls Telegram Bot API `banChatMember`.
- **Config:** POST to `https://api.telegram.org/YOUR_TELEGRAM_BOT_TOKE/banChatMember` with `chat_id` and `user_id`.
- **Risk/edge cases:**
  - URL contains placeholder token and a likely typo (`YOUR_TELEGRAM_BOT_TOKE` missing “N”).
  - Bot must be admin with ban permissions.
  - Telegram may require `unban` for temporary ban; this call is a ban, not a kick.

---

### Block 2.3 — Membership Gating (Message Flow)
**Overview:** Ensures the user is a member of a required channel/group before providing results.

**Nodes involved:**
- Checking if we are in a certain group
- If2
- Request to join the group

#### Checking if we are in a certain group
- **Type / role:** `telegram` chat member check.
- **Config:**
  - `operation: member`, `resource: chat`
  - `chatId: @your group` (placeholder)
  - `userId: Telegram Trigger.message.from.id`
- **Output:** Typically includes `result.status` (e.g., member, left, kicked).
- **Failure types:** Bot not admin in target channel; wrong chat username; Telegram API errors.

#### If2
- **Type / role:** `if` — gates by membership.
- **Config:** `$json.result.status != "left"` (true = member/allowed).
- **Outputs:**
  - **True:** proceed to wording validation (`If4`).
  - **False:** send join prompt (`Request to join the group`).
- **Edge cases:** Status can be `restricted`, `kicked`, `administrator`, etc. This logic only blocks `left` and would allow `kicked` unless Telegram uses different status value.

#### Request to join the group
- **Type / role:** `telegram` — prompt user to join.
- **Config:** Inline keyboard button with URL: `https://t.me/aliexpressdils`.
- **Reply:** to the user’s message.
- **Edge cases:** User in private chat vs group; link/button works but membership still not refreshed until next message.

---

### Block 2.4 — Starter Phrase Validation (Message Flow)
**Overview:** Requires the message to start with one of several Hebrew starter phrases; otherwise instructs correct format.

**Nodes involved:**
- If4
- Request for correct wording
- opening message

#### If4
- **Type / role:** `if` string prefix checks.
- **Config:** OR of `startsWith` against:
  - “תמצא לי”, “תחפש לי”, “חפש לי”, “מצא לי”, “תשלח לי”, “שלח לי”
- **Input:** Membership-approved path.
- **Outputs:**
  - **True:** `opening message` (progress message).
  - **False:** `Request for correct wording`.

#### Request for correct wording
- **Type / role:** `telegram` — instructs user how to phrase request.
- **Config:** Example includes the user’s text; replies to original message.

#### opening message
- **Type / role:** `telegram` — sends “Just a moment…” placeholder.
- **Output:** Used later for deletion (`Deleting a message`).

---

### Block 2.5 — Search Query Cleanup + Scrape + Extract (Message Flow)
**Overview:** Converts the user request to a clean AliExpress search term, scrapes AliExpress result page HTML via Decodo, and extracts product objects.

**Nodes involved:**
- Start typing
- Creating a professional search term
- data scraping
- Follow up message
- Extract all items
- Extraction of 2 products from the first 10

#### Start typing
- **Type / role:** `telegram` sendChatAction.
- **Config:** Uses chatId from `opening message.result.chat.id`.

#### Creating a professional search term
- **Type / role:** OpenAI node — cleans query (Hebrew prompt).
- **Config:**
  - Input: `Telegram Trigger.message.text.split(' ').slice(2).join(' ')` (drops first two tokens).
  - Returns only cleaned product name.
- **Edge cases:** Assumes at least 3 tokens; different starter phrase lengths; may cut too much/too little.

#### data scraping
- **Type / role:** `@decodo/n8n-nodes-decodo.decodo` — fetches AliExpress search page HTML.
- **Config:**
  - GEO: Israel
  - URL: `https://aliexpress.com/w/wholesale-{{ cleanedTerm.replaceAll(' ','-') }}.html`
  - Retry: maxTries 5, wait 5s, `retryOnFail: true`
- **Failure types:** AliExpress blocking, bot detection, Decodo quota, HTML layout changes.

#### Follow up message
- **Type / role:** `telegram` — sends “I have everything…” progress message.
- **executeOnce:** true (prevents duplicates per execution branch).

#### Extract all items
- **Type / role:** `code` — intended to parse embedded JSON from scripts.
- **Important:** The provided code is **incomplete/broken** in JSON (missing declarations like `scripts`, `products`, and contains invalid tokens like `to continue;`, truncated try/catch, unfinished parsing). As-is, it will likely fail at runtime.
- **Expected role:** Return an array of product objects each with fields like `productId`, `title`, `mainImage`, `salePrice`, etc.
- **Edge cases:** AliExpress frequently changes embedded data keys; parsing must be robust.

#### Extraction of 2 products from the first 10
- **Type / role:** `code` — selects first 2 products and constructs canonical item URLs.
- **Config:**
  - Normalizes input as either array-in-one-item or multiple items.
  - `cleanUrl = https://www.aliexpress.com/item/${product.productId}.html`
- **Output:** Two items, each with `productUrl` added.
- **Edge cases:** Missing `productId`; product list shorter than 2.

---

### Block 2.6 — Affiliate Link Generation + Retry Path (Message Flow)
**Overview:** Generates affiliate links for the two selected products. If affiliate generation fails for all items, the flow loops back to re-extract items (fallback).

**Nodes involved:**
- Creating an affiliate link3
- Code in JavaScript
- whether there is an error or not
- Wording for message 3
- (fallback output of whether there is an error or not → Extract all items)

#### Creating an affiliate link3
- **Type / role:** `n8n-nodes-aliexpress-affiliate.aliExpressAffiliate`
- **Config:**
  - `tracking_id: YOUR_AFFILIATE_TRACKING_ID`
  - `source_values: {{$json.productUrl}}`
  - `promotion_link_type: 2`
- **Retry:** enabled.
- **Failure types:** invalid credentials, tracking id, API quota; AliExpress rejects URL.

#### Code in JavaScript
- **Type / role:** `code` — checks if at least one item succeeded (resp_code 200).
- **Logic:** If any success → return all items; else return `{result: 2}` sentinel.
- **Edge cases:** Response path mismatch; partial successes.

#### whether there is an error or not
- **Type / role:** `if` — checks sentinel.
- **Config:** `$json.result != 2` (true = ok).
- **Outputs:**
  - **True:** proceed to “Wording for message 3”.
  - **False:** fallback to “Extract all items” (re-attempt extraction and selection chain).

---

### Block 2.7 — Compose & Send Product Card + Cleanup (Message Flow)
**Overview:** Generates a persuasive Hebrew caption and sends a photo message to Telegram with inline buttons: “more results” callback and “purchase” affiliate URL. Deletes temporary progress messages.

**Nodes involved:**
- Wording for message 3
- sending a message
- Deleting a message
- Deleting a message 1

#### Wording for message 3
- **Type / role:** OpenAI — copywriting in Hebrew.
- **Config:** Uses product fields from “Extraction of 2 products…” and user query from Telegram Trigger message.
- **Output:** `choices[0].message.content` used as caption.
- **Edge cases:** If OpenAI returns unexpected structure; if product fields missing.

#### sending a message
- **Type / role:** `telegram` sendPhoto.
- **Config:**
  - `file`: product `mainImage`
  - Inline keyboard:
    - `callback_data: "more_results"`
    - purchase URL from affiliate response:
      `...promotion_link[0].promotion_link`
  - Caption: OpenAI output; parse_mode HTML
  - `onError: continueRegularOutput` + `alwaysOutputData: true`
- **Edge cases:** Telegram rejects remote image URL; HTML parsing issues; affiliate link missing.

#### Deleting a message / Deleting a message 1
- **Type / role:** `telegram` deleteMessage
- **Config:** Deletes `opening message` and `Follow up message` placeholders.
- **Requires:** admin delete permissions.

---

### Block 2.8 — Callback Query (“More Results”) Security + Membership + Wording Validation
**Overview:** Handles user clicking “You will find more”. Ensures the clicker is the original requester and is a channel member, validates the original message starter phrase, then proceeds with alternate selection logic (random from items 3–7).

**Nodes involved:**
- Only if the person who asked for more is the one who sent the original message
- A message that only those who sent an original message can request more
- Wait
- Deleting a message 5
- Checking if we are in a certain group 
- If3
- Request to join the group 1
- If5
- Request for correct wording 1
- Opening message 2

#### Only if the person who asked for more is the one who sent the original message
- **Type / role:** `if` — authorization check.
- **Config:** Compares:
  - `callback_query.message.reply_to_message.from.id`
  - vs `callback_query.from.id`
- **Outputs:**
  - **True:** proceed to membership check (“Checking if we are in a certain group ”).
  - **False:** send lock message (“A message that only…”).
- **Edge cases:** If `reply_to_message` missing (e.g., bot message not replying); expression errors.

#### A message that only those who sent an original message can request more
- **Type / role:** `telegram` — informs unauthorized clicker.
- **Then:** Wait → Deleting a message 5 (auto-cleanup).
- **Config:** Replies to the original user message id; includes a lock text.

#### Wait
- **Type / role:** `wait` — delays then continues (used for timed deletion).
- **Note:** No explicit duration set in parameters; default behavior depends on n8n version (often needs “wait time” or “resume webhook”). As configured, may not behave as intended.

#### Deleting a message 5
- **Type / role:** `telegram` deleteMessage
- **Config:** Deletes the lock message that was just sent.

#### Checking if we are in a certain group  (with trailing space)
- **Type / role:** `telegram` member check for callback user.
- **Config:** `userId = callback_query.from.id`.
- **Output:** `result.status`.

#### If3
- **Type / role:** `if` membership gate for callback path.
- **Config:** `$json.result.status != "left"`.
- **Outputs:** True → If5 ; False → Request to join the group 1.

#### Request to join the group 1
- **Type / role:** `telegram` join prompt (same idea as message flow).
- **Config:** Sends to `message.chat.id` (note: uses `message` path; in callback flows this may be inconsistent depending on trigger payload—here it references `Telegram Trigger.message.chat.id`, which might not exist for callback updates).

#### If5
- **Type / role:** `if` starter phrase validation using `callback_query.message.reply_to_message.text`.
- **Outputs:** True → Opening message 2; False → Request for correct wording 1.

#### Opening message 2 / Request for correct wording 1
- **Type / role:** Telegram messages similar to message flow, but replying to `reply_to_message.message_id`.

---

### Block 2.9 — Callback Query: Scrape + Extract + Random Pick + Affiliate + Send + Cleanup
**Overview:** Performs the same search/scrape process but selects 2 random items from positions 3–7, then sends a new product card as a reply to the original request message. Deletes progress messages afterward.

**Nodes involved:**
- Start typing 1
- Creating a professional search term 1
- Data scraping1
- Follow-up message 1
- Extracting all items 1
- Extraction of 2 products from the first 7
- Creating an affiliate link
- Code in JavaScript2
- If there is an error or not 1
- Wording for message
- Sending a message 1
- Deleting a message 3
- Deleting a message4

Key differences vs message flow:
- **Creating a professional search term 1** uses an English system prompt but same “split().slice(2)” extraction.
- **Extraction of 2 products from the first 7** skips first 2 and randomly selects 2 from next 5.
- Cleanup deletes `Opening message 2` and `Follow-up message 1`.

**Important note:** `Extracting all items 1` code is also visibly corrupted (contains Hebrew keywords and truncated JS). It will need repair for the workflow to run reliably.

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram message/callback | — | Bot start filtering | 📥 Input Handler<br>Receives incoming messages and callback queries from Telegram.<br><br># AliExpress Affiliate Bot for Telegram<br>Automatically find and share AliExpress products with affiliate links in your Telegram group.<br><br>## How it works<br>1. Bot receives messages and callback queries from Telegram group<br>2. AI moderator checks content for spam, links, and inappropriate requests<br>3. Valid product requests trigger an AliExpress search<br>4. System generates affiliate tracking links for top matching products<br>5. Formatted product cards are sent back to the Telegram group<br>6. Users can request additional options via inline buttons<br><br>## Setup steps<br>1. Create a Telegram Bot via @BotFather and obtain API token<br>2. Register for AliExpress Affiliate Program and get App Key & Secret<br>3. Create OpenAI API account and generate API key<br>4. Add all three credentials in n8n (Telegram, AliExpress, OpenAI)<br>5. Replace placeholder credential IDs with your own<br>6. Add the bot to your Telegram group with admin permissions<br>7. Activate the workflow and test with a sample product request |
| Bot start filtering | filter | Ignore /start | Telegram Trigger | If | 📥 Input Handler… (same as above) |
| If | if | Route message vs callback | Bot start filtering | Message a model; Only if the person… | 📥 Input Handler… (same as above) |
| Message a model | OpenAI (LangChain) | Moderation classification | If | Code in JavaScript1 | 🛡️ Content Moderation<br>AI validates message content. Blocks spam, links, and inappropriate requests. |
| Code in JavaScript1 | code | Parse moderation JSON | Message a model | If there is problematic content | 🛡️ Content Moderation… |
| If there is problematic content | if | Approved vs rejected routing | Code in JavaScript1 | Checking if we are in a certain group; Opening message 1 | 🛡️ Content Moderation… |
| Opening message 1 | telegram | Send rejection notice | If there is problematic content (false) | Deleting a message 2 | 🛡️ Content Moderation… |
| Deleting a message 2 | telegram | Delete offending user message | Opening message 1 | Automatic removal of the user from the group | 🛡️ Content Moderation… |
| Automatic removal of the user from the group | httpRequest | Ban/kick user via Telegram API | Deleting a message 2 | — | 🛡️ Content Moderation… |
| Checking if we are in a certain group | telegram | Membership check (message sender) | If there is problematic content (true) | If2 |  |
| If2 | if | Gate by membership | Checking if we are in a certain group | If4; Request to join the group |  |
| Request to join the group | telegram | Prompt to join required channel | If2 (false) | — |  |
| If4 | if | Validate starter phrases (message) | If2 (true) | opening message; Request for correct wording |  |
| Request for correct wording | telegram | Instruct correct phrasing | If4 (false) | — |  |
| opening message | telegram | Progress placeholder | If4 (true) | Start typing | 🔍 Product Search<br>Translates request to search query and scrapes AliExpress for products. |
| Start typing | telegram | Chat action typing | opening message | Creating a professional search term | 🔍 Product Search… |
| Creating a professional search term | OpenAI (LangChain) | Clean/optimize query | Start typing | data scraping | 🔍 Product Search… |
| data scraping | Decodo | Scrape AliExpress HTML | Creating a professional search term | Follow up message | 🔍 Product Search… |
| Follow up message | telegram | Progress placeholder | data scraping | Extract all items | 🔍 Product Search… |
| Extract all items | code | Parse products from HTML scripts | Follow up message | Extraction of 2 products from the first 10 | 🔍 Product Search… |
| Extraction of 2 products from the first 10 | code | Select first 2 products | Extract all items | Creating an affiliate link3 | 🔗 Affiliate Link Generation<br>Creates tracking links via AliExpress API. Handles errors gracefully. |
| Creating an affiliate link3 | AliExpress Affiliate | Generate affiliate links | Extraction of 2 products from the first 10 | Code in JavaScript | 🔗 Affiliate Link Generation… |
| Code in JavaScript | code | Detect affiliate success/sentinel | Creating an affiliate link3 | whether there is an error or not | 🔗 Affiliate Link Generation… |
| whether there is an error or not | if | Route success vs retry | Code in JavaScript | Wording for message 3; Extract all items | 🔗 Affiliate Link Generation… |
| Wording for message 3 | OpenAI (LangChain) | Write Hebrew sales caption | whether there is an error or not (true) | sending a message | 📤 Response Handler<br>Sends product cards to Telegram and cleans up temporary messages. |
| sending a message | telegram | Send product photo + buttons | Wording for message 3 | Deleting a message | 📤 Response Handler… |
| Deleting a message | telegram | Delete “Just a moment…” | sending a message | Deleting a message 1 | 📤 Response Handler… |
| Deleting a message 1 | telegram | Delete follow-up placeholder | Deleting a message | — | 📤 Response Handler… |
| Only if the person who asked for more is the one who sent the original message | if | Callback authorization | If (callback branch) | Checking if we are in a certain group ; A message that only… |  |
| A message that only those who sent an original message can request more | telegram | Warn unauthorized clicker | Only if… (false) | Wait |  |
| Wait | wait | Delay before cleanup | A message that only… | Deleting a message 5 |  |
| Deleting a message 5 | telegram | Delete warning message | Wait | — |  |
| Checking if we are in a certain group  | telegram | Membership check (callback user) | Only if… (true) | If3 |  |
| If3 | if | Gate callback by membership | Checking if we are in a certain group  | If5; Request to join the group 1 |  |
| Request to join the group 1 | telegram | Prompt join (callback path) | If3 (false) | — |  |
| If5 | if | Validate starter phrases (reply_to_message.text) | If3 (true) | Opening message 2; Request for correct wording 1 |  |
| Request for correct wording 1 | telegram | Instruct phrasing (callback path) | If5 (false) | — |  |
| Opening message 2 | telegram | Progress placeholder (callback) | If5 (true) | Start typing 1 | 🔍 Product Search… |
| Start typing 1 | telegram | Chat action typing (callback) | Opening message 2 | Creating a professional search term 1 | 🔍 Product Search… |
| Creating a professional search term 1 | OpenAI (LangChain) | Clean/optimize query (callback) | Start typing 1 | Data scraping1 | 🔍 Product Search… |
| Data scraping1 | Decodo | Scrape AliExpress HTML (callback) | Creating a professional search term 1 | Follow-up message 1 | 🔍 Product Search… |
| Follow-up message 1 | telegram | Progress placeholder (callback) | Data scraping1 | Extracting all items 1 | 🔍 Product Search… |
| Extracting all items 1 | code | Parse products (callback) | Follow-up message 1 | Extraction of 2 products from the first 7 | 🔍 Product Search… |
| Extraction of 2 products from the first 7 | code | Random 2 from items 3–7 | Extracting all items 1 | Creating an affiliate link | 🔗 Affiliate Link Generation… |
| Creating an affiliate link | AliExpress Affiliate | Affiliate links (callback) | Extraction of 2 products from the first 7 | Code in JavaScript2 | 🔗 Affiliate Link Generation… |
| Code in JavaScript2 | code | Detect affiliate success/sentinel | Creating an affiliate link | If there is an error or not 1 | 🔗 Affiliate Link Generation… |
| If there is an error or not 1 | if | Route success vs retry | Code in JavaScript2 | Wording for message; Extracting all items 1 | 🔗 Affiliate Link Generation… |
| Wording for message | OpenAI (LangChain) | Sales caption (callback) | If there is an error or not 1 (true) | Sending a message 1 | 📤 Response Handler… |
| Sending a message 1 | telegram | Send product card (callback) | Wording for message | Deleting a message 3 | 📤 Response Handler… |
| Deleting a message 3 | telegram | Delete Opening message 2 | Sending a message 1 | Deleting a message4 | 📤 Response Handler… |
| Deleting a message4 | telegram | Delete Follow-up message 1 | Deleting a message 3 | — | 📤 Response Handler… |
| If4 / If2 / If3 / If5 / If there is… nodes | if | Routing/gating | various | various | (covered above) |
| Sticky Note / Sticky Note1..5 | stickyNote | Documentation only | — | — | (N/A) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1) **Telegram API** credential: add bot token from @BotFather.  
   2) **OpenAI API** credential: add API key with access to the selected model (`gpt-5.2` as used here).  
   3) **Decodo** credential: configure the Decodo API key/account in n8n.  
   4) **AliExpress Affiliate** credential: configure AliExpress affiliate App Key/Secret (per the node package requirements) and ensure you have a valid `tracking_id`.

2. **Create the trigger**
   - Add **Telegram Trigger** node:
     - Updates: `message`, `callback_query`.

3. **Add `/start` filter**
   - Add **Filter** node “Bot start filtering”:
     - Condition: `message.text != "/start"`.
   - Connect: Telegram Trigger → Bot start filtering.

4. **Split message vs callback**
   - Add **IF** node “If”:
     - Condition: `callback_query.data` **not exists**.
   - Connect: Bot start filtering → If.
   - True output = message flow; False output = callback flow.

5. **Message flow: OpenAI moderation**
   - Add **OpenAI (LangChain)** node “Message a model”:
     - User content: `{{$json.message.text}}`
     - System prompt: implement the Hebrew JSON moderation rules (as in the workflow).
   - Add **Code** node “Code in JavaScript1” to parse JSON:
     - Read model output text, strip markdown fences, `JSON.parse`.
   - Add **IF** node “If there is problematic content”:
     - Condition: `status != "rejected"`.
   - Connect: If (true/message) → Message a model → Code in JavaScript1 → If there is problematic content.

6. **Rejected branch: notify + delete + ban**
   - Add **Telegram** node “Opening message 1” (sendMessage):
     - `text: {{$json.message}}`
     - `chatId: {{$node["Telegram Trigger"].json.message.chat.id}}`
   - Add **Telegram** node “Deleting a message 2” (deleteMessage):
     - `chatId` as above
     - `messageId: {{$node["Telegram Trigger"].json.message.message_id}}`
   - Add **HTTP Request** node “Automatic removal…”:
     - POST `https://api.telegram.org/<BOT_TOKEN>/banChatMember`
     - Body params: `chat_id`, `user_id`.
   - Connect: If there is problematic content (false) → Opening message 1 → Deleting a message 2 → Automatic removal…

7. **Approved branch: membership check (message sender)**
   - Add **Telegram** node “Checking if we are in a certain group” (chat member):
     - `chatId: @your group` (replace with required channel/group username or ID)
     - `userId: {{$node["Telegram Trigger"].json.message.from.id}}`
   - Add **IF** node “If2”:
     - Condition: `result.status != "left"`.
   - Add **Telegram** node “Request to join the group” (sendMessage):
     - Include inline keyboard URL button to your channel.
   - Connect: Approved → Checking membership → If2; If2 false → Request to join the group.

8. **Message flow: validate starter phrase**
   - Add **IF** node “If4” with OR `startsWith` checks for the Hebrew starters.
   - Add **Telegram** node “Request for correct wording” if invalid.
   - Add **Telegram** node “opening message” (progress) if valid.
   - Connect: If2 true → If4; If4 false → Request for correct wording; If4 true → opening message.

9. **Message flow: query cleanup, scrape, extract, select**
   - Add **Telegram** node “Start typing” (sendChatAction) → connect from opening message.
   - Add **OpenAI** node “Creating a professional search term”:
     - Input extraction expression: split the message and drop starter tokens (as used).
   - Add **Decodo** node “data scraping”:
     - URL: `https://aliexpress.com/w/wholesale-<term-with-dashes>.html`
     - retries enabled.
   - Add **Telegram** node “Follow up message”.
   - Add **Code** node “Extract all items”:
     - Implement robust JS to parse AliExpress embedded JSON and return product objects.
     - (You will need to fix/replace the broken code from the provided workflow.)
   - Add **Code** node “Extraction of 2 products from the first 10”:
     - Take first 2; add `productUrl`.
   - Connect: Start typing → OpenAI term → Decodo → Follow up → Extract all items → Extraction first 2.

10. **Message flow: affiliate link generation with fallback**
   - Add **AliExpress Affiliate** node “Creating an affiliate link3” using `productUrl`.
   - Add **Code** node “Code in JavaScript” to check `resp_code === 200` in any item; else output `{result:2}`.
   - Add **IF** node “whether there is an error or not”:
     - Condition: `result != 2`.
     - False branch loops back to “Extract all items”.
   - Connect: Extraction → Affiliate → Code check → IF → (true continue, false loop).

11. **Message flow: caption + send photo + cleanup**
   - Add **OpenAI** node “Wording for message 3” (Hebrew sales copy).
   - Add **Telegram** node “sending a message” (sendPhoto):
     - `file` = product `mainImage`
     - caption from OpenAI
     - inline keyboard:
       - callback button `more_results`
       - URL button = affiliate promotion link
   - Add **Telegram** delete nodes to remove progress messages.
   - Connect: IF success → Wording → sendPhoto → delete opening message → delete follow-up.

12. **Callback flow: authorization, membership gate, starter validation**
   - From If (false/callback), add **IF** “Only if the person…” comparing `reply_to_message.from.id` vs `callback_query.from.id`.
   - False branch: send warning message → Wait → delete warning.
   - True branch: membership check node (chat member) for `callback_query.from.id` → If3 membership gate → If5 starter phrase check on `reply_to_message.text`.

13. **Callback flow: scrape + extract + random selection**
   - Mirror the message flow nodes but:
     - Use “Opening message 2” + “Start typing 1”.
     - Use “Extraction of 2 products from the first 7” (random 2 from slice(2,7)).
   - Affiliate generation: “Creating an affiliate link” + “Code in JavaScript2” + “If there is an error or not 1” with fallback loop to extraction.
   - Send result: “Sending a message 1” replying to the original request message, then delete “Opening message 2” and “Follow-up message 1”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Join-gating channel link used in buttons | https://t.me/aliexpressdils |
| Workflow assumes bot has admin permissions in the Telegram group (delete messages, ban members) | Telegram group/supergroup administration |
| Two HTML-parsing Code nodes appear corrupted/incomplete and must be fixed for production | Nodes: “Extract all items”, “Extracting all items 1” |
| Telegram ban endpoint URL contains placeholders and likely a typo; must be replaced with real bot token | `https://api.telegram.org/<BOT_TOKEN>/banChatMember` |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.