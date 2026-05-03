Create Shopify products via a Telegram bot with Google Gemini AI

https://n8nworkflows.xyz/workflows/create-shopify-products-via-a-telegram-bot-with-google-gemini-ai-13333


# Create Shopify products via a Telegram bot with Google Gemini AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** A Telegram bot guides a user through submitting product info (name, images, features, prices), uses **Google Gemini** to generate ecommerce copy (long description, short description, slug) and image metadata (title/alt tags), then **creates a product in Shopify** via HTTP requests and notifies the user.

**Target use cases:**
- Semi-automated product creation for Shopify from a mobile-friendly Telegram chat flow.
- AI-assisted content generation and image alt-tagging for accessibility/SEO.
- Stateful multi-step chatbot using n8n Data Tables to track each user’s progress.

### Logical blocks
1.1 **Telegram Intake & User State Lookup** → loads user state, supports `/abort`, routes based on state.  
1.2 **Start / Restart Handling** → `/start` gate, stores chat id, asks for product name.  
1.3 **Conversational Product Data Collection** → step machine: product name → images upload → “done” → features → regular price → sale price → approval.  
1.4 **AI Content Generation (Gemini)** → generates descriptions + slugs (two mirrored branches).  
1.5 **Image Processing & AI Alt/Title Generation** → fetches images, uses Gemini agent + structured parser, maps alt tags to URLs, downloads binaries, base64 encodes.  
1.6 **Shopify Product Creation & Notifications** → sends payload to Shopify, notifies success, proposes creating another product, resets DBs.

---

## 2. Block-by-Block Analysis

### 1.1 Telegram Intake & User State Lookup

**Overview:** Receives all Telegram updates, retrieves the sender’s state from a Data Table, and branches for abort vs continuing the flow.

**Nodes involved:**
- Telegram Trigger
- Get User State
- If /abort
- Reset Shopify Product Manager DB
- Reset User_images DB
- Notifying user about process abortion
- Fetch Data from User State DB
- If3

#### Telegram Trigger
- **Type/role:** `telegramTrigger` — entry point, listens to Telegram bot updates.
- **Config choices:** Not shown in JSON (empty parameters). In practice must be configured with Telegram credentials and update types (message, photo, callback_query, etc.).
- **I/O:** Output → **Get User State**
- **Failures:** Telegram credential invalid; webhook/polling misconfigured; missing permissions for group updates.

#### Get User State
- **Type/role:** `dataTable` — reads current user state (step, chat id, etc.).
- **Config:** Not shown; `alwaysOutputData: true` suggests downstream logic expects output even when no row exists.
- **I/O:** Input from Trigger → Output to **If /abort**
- **Edge cases:** No row found; schema mismatch; Data Table not created.

#### If /abort
- **Type/role:** `if` — checks whether user sent `/abort` (or an abort signal).
- **Config:** Conditions not visible; expected to inspect Telegram message text.
- **I/O:**
  - True → **Reset Shopify Product Manager DB** (then Reset User_images DB → notify abortion)
  - False → **Fetch Data from User State DB**
- **Failures:** Expression errors if Telegram payload path differs (e.g., callback query vs message).

#### Reset Shopify Product Manager DB
- **Type/role:** `dataTable` — clears per-user product manager state.
- **I/O:** → **Reset User_images DB**
- **Edge cases:** If clearing globally instead of per-user, could wipe all users.

#### Reset User_images DB
- **Type/role:** `dataTable` — clears stored images state.
- **Config:** `alwaysOutputData: true` (ensures notify step runs).
- **I/O:** → **Notifying user about process abortion**
- **Failures:** same as above.

#### Notifying user about process abortion
- **Type/role:** `telegram` — sends message confirming the flow was aborted.
- **Config:** Not shown (chat id expression expected).
- **I/O:** terminal in this branch.
- **Failures:** chat id missing; bot blocked by user.

#### Fetch Data from User State DB
- **Type/role:** `code` — normalizes/derives routing fields from state (e.g., current step).
- **Config:** Not visible; likely reads Data Table row(s) and sets something like `currentStep`.
- **I/O:** → **If3**
- **Failures:** JavaScript errors; missing fields in state row.

#### If3
- **Type/role:** `if` — determines whether a user state exists or if the flow should go to `/start` gate.
- **I/O:**
  - True → **Get User State1**
  - False → **Check for /start**
- **Edge cases:** brand-new users; state row exists but incomplete.

---

### 1.2 Start / Restart Handling

**Overview:** Ensures the user begins with `/start`, captures chat id, and prompts for the product name.

**Nodes involved:**
- Check for /start
- Prompting user for product name
- Chat ID Update
- Prompting user to use /Start
- Get User State1

#### Check for /start
- **Type/role:** `if` — checks if message is `/start`.
- **I/O:**
  - True → **Prompting user for product name**
  - False → **Prompting user to use /Start**
- **Edge cases:** `/start@YourBot` in groups; localized commands.

#### Prompting user for product name
- **Type/role:** `telegram` — asks user to provide product name.
- **I/O:** → **Chat ID Update**
- **Failures:** cannot message user; incorrect chat id mapping.

#### Chat ID Update
- **Type/role:** `dataTable` — upserts chat id and initializes state/step.
- **I/O:** terminal here; subsequent messages re-enter via Trigger and state routing.
- **Edge cases:** multiple users; need a stable key (telegram user id).

#### Prompting user to use /Start
- **Type/role:** `telegram` — instructs user to send `/start`.
- **I/O:** terminal.

#### Get User State1
- **Type/role:** `dataTable` — fetches state again for known users.
- **Config:** `alwaysOutputData: true`.
- **I/O:** → **Switch**
- **Edge cases:** stale state, invalid step value.

---

### 1.3 Conversational Product Data Collection (State Machine)

**Overview:** A `Switch` routes the conversation based on the saved “current step”. The user uploads images, sends “done”, supplies features and prices, then approves creation.

**Nodes involved:**
- Switch
- Check Approval Button
- Notifying user about Product Creation Initiation
- Updating Current Step
- Fetching Product Data4
- Code in JavaScript
- Edit Fields
- Code in JavaScript2
- HTTP Request
- If2
- Updating Current Step & Product Name
- Notifying User to Specify Product Name
- Check for Done after uploading images
- Code
- Get a file
- Edit Fields1
- Updating Image Data
- Fetching Product Data5
- Combining File IDs & URLs
- Updating Current Step
- Prompting user for Features
- Check for None in Features
- Updating Current Step1
- Prompting user for Regular Price
- Updating Current Step & Features
- Prompting user for Regular Price1
- Updating Current Step2
- Prompting user for Sale Price
- If5
- Updating Sale Price
- Updating Sale Price1
- Merge
- Get All Data
- Code in JavaScript4
- If6
- Telegram Group Media
- Prompting user for Approval
- Prompting user to create another product
- Reset Shopify Product Manager DB1
- Reset User_Images DB
- Notifying User to Restart
- Upsert row(s)1
- Prompting user for product name (already covered in 1.2)
- Notifying user about Product Creation Initiation (covered here)

#### Switch
- **Type/role:** `switch` — routes by the stored step (e.g., “awaiting_name”, “awaiting_images”, “awaiting_features”, etc.).
- **Outputs:** 7 outputs wired to:
  1) Check Approval Button  
  2) Updating Current Step & Product Name  
  3) Check for Done after uploading images  
  4) Check for None in Features  
  5) Updating Current Step2  
  6) If5  
  7) Prompting user to create another product
- **Edge cases:** step value not matching any case → dropped execution (unless configured with “fallback”).

#### Check Approval Button
- **Type/role:** `if` — checks if user clicked approval (likely callback data).
- **I/O:** 
  - True → Notifying user about Product Creation Initiation
  - False → Upsert row(s)1 (then Notifying User to Restart)
- **Edge cases:** approvals via text vs inline keyboard; callback payload path differs.

#### Upsert row(s)1 → Notifying User to Restart
- **Role:** Resets/sets state when approval flow is inconsistent, then asks user to restart.
- **Failure modes:** Data Table upsert conflicts; missing user key.

#### Updating Current Step → Fetching Product Data4 → Code in JavaScript → Edit Fields → Code in JavaScript2 → HTTP Request → If2
This chain appears to prepare some product payload and decide which AI branch to use.
- **HTTP Request:** purpose unclear from parameters (empty), but placed before **If2**; likely fetches something (maybe product manager row, images, or a template).
- **If2:** routes to **Product Description Generation** (branch A) or **Product Description Generation1** (branch B). This suggests two modes (e.g., with/without features, or different languages).

#### Updating Current Step & Product Name → Notifying User to Specify Product Name
- **Role:** persists the provided product name and moves to next step, then tells user what to do next.
- **Edge cases:** empty name, very long name, unsupported characters.

#### Check for Done after uploading images
- **Type/role:** `if` — checks whether user wrote “done” (or similar).
- **I/O:**
  - True → Fetching Product Data5 (then combining IDs/URLs and prompting for features)
  - False → Code (then Get a file → store image data)
- **Edge cases:** user sends media albums; Telegram updates can be `photo[]`, `document`, etc.

#### Code → Get a file → Edit Fields1 → Updating Image Data
- **Role:** extracts Telegram file_id from incoming message; downloads file metadata/binary reference; normalizes fields; stores image reference in Data Table.
- **Get a file:** Telegram node that calls `getFile` API.
- **Failures:** file too large; Telegram rate limits; missing `file_id` path.

#### Fetching Product Data5 → Combining File IDs & URLs → Updating Current Step  → Prompting user for Features
- **Role:** after “done”, compiles all uploaded images into a list (file IDs and URLs), stores, advances step, asks for features.
- **Edge cases:** no images uploaded but user sends “done”; mismatch between file IDs and URLs.

#### Check for None in Features
- **Type/role:** `if` — if user says “none” (no features), skip storing features.
- **I/O:**
  - True → Updating Current Step1 → Prompting user for Regular Price
  - False → Updating Current Step & Features → Prompting user for Regular Price1
- **Edge cases:** user enters “None”, “N/A”, localized equivalents.

#### Updating Current Step2 → Prompting user for Sale Price
- **Role:** saves regular price, advances to sale price step.
- **Edge cases:** invalid numeric format; currency symbols; decimal commas.

#### If5 → Updating Sale Price / Updating Sale Price1 → Merge
- **Role:** checks sale price presence/validity (two branches), updates Data Table accordingly, then merges both paths.
- **Merge:** `merge` combines items from both branches.
- **Edge cases:** Merge mode not visible; wrong mode can duplicate or drop.

#### Merge → Get All Data → Code in JavaScript4 → If6 → Telegram Group Media → Prompting user for Approval
- **Role:** collects final dataset, formats it, optionally posts product preview media to a Telegram group/channel (`Telegram Group Media`), then prompts user for approval.
- **If6:** gates whether to send group media.
- **Edge cases:** bot not in group; group id wrong; sending media requires correct payload.

#### Prompting user to create another product → Reset Shopify Product Manager DB1 → Reset User_Images DB
- **Role:** after completion, asks if user wants another product, then resets state and images tables for a fresh run.
- **Edge cases:** ensure per-user reset, not global.

---

### 1.4 AI Content Generation (Gemini)

**Overview:** Generates product long description, short description, and slug. There are **two parallel pipelines** (A and B), likely for different product types or different stored inputs.

**Nodes involved (Pipeline A):**
- Product Description Generation
- Google Gemini Chat Model
- Markdown
- Short Description Generation
- Google Gemini Chat Model1
- Slug Generation
- Google Gemini Chat Model3
- Fetching Product Data2 → Fetching Image URLs2 → Getting Image Data1 → Image Title & Alt Tag Creation1 ... (hands off to image block)

**Nodes involved (Pipeline B):**
- Product Description Generation1
- Google Gemini Chat Model2
- Markdown1
- Short Description Generation1
- Google Gemini Chat Model4
- Slug Generation1
- Google Gemini Chat Model5
- Fetching Product Data → Fetching Image URLs → Getting Image Data → Image Title & Alt Tag Creation ... (hands off to image block)

#### Product Description Generation / Product Description Generation1
- **Type/role:** `chainLlm` — LLM chain prompt to generate rich product description.
- **Model binding:** via `ai_languageModel` input from a Gemini Chat Model node.
- **I/O:** → Markdown / Markdown1
- **Edge cases:** hallucinated claims; policy-sensitive categories; overly long output.

#### Markdown / Markdown1
- **Type/role:** `markdown` — formats/cleans description output (e.g., HTML or Markdown normalization).
- **I/O:** → Short Description Generation / Short Description Generation1
- **Edge cases:** invalid markdown, unexpected model formatting.

#### Short Description Generation / Short Description Generation1
- **Type/role:** `chainLlm` — generates concise excerpt.
- **I/O:** → Slug Generation / Slug Generation1

#### Slug Generation / Slug Generation1
- **Type/role:** `chainLlm` — generates URL slug.
- **Edge cases:** spaces, uppercase, diacritics; must be normalized to Shopify handle rules.

#### Google Gemini Chat Model nodes (0–5)
- **Type/role:** `lmChatGoogleGemini`
- **Config:** Not visible; must set:
  - Model (e.g., `gemini-1.5-pro`, `gemini-1.5-flash`)
  - API key / credentials
  - Temperature, max tokens (optional)
- **Failure modes:** invalid API key; quota exceeded; safety blocks; timeouts.

---

### 1.5 Image Processing & AI Alt/Title Generation

**Overview:** Fetches stored image URLs, downloads image data, asks Gemini agent to produce **image title + alt tag** in structured form, maps back to URLs, downloads binaries and base64-encodes for Shopify API.

**Nodes involved (Branch B / “Generation1” path):**
- Fetching Product Data
- Fetching Image URLs
- Getting Image Data
- Image Title & Alt Tag Creation
- Google Gemini Chat Model7
- Structured Output Parser
- Mapping Image URL with Alt Tag
- Fetching Product Data1
- Fetching Image URLs1
- Fetching Binary Data
- Formatting Images as Base64

**Nodes involved (Branch A / “Generation” path):**
- Fetching Product Data2
- Fetching Image URLs2
- Getting Image Data1
- Image Title & Alt Tag Creation1
- Google Gemini Chat Model8
- Structured Output Parser1
- Mapping Image URL with Alt Tag1
- Fetching Product Data3
- Fetching Image URLs3
- Fetching Binary Data1
- Formatting Images as Base64_1

#### Fetching Product Data / Fetching Product Data2 / Fetching Product Data1 / Fetching Product Data3
- **Type/role:** `dataTable` — read product row containing image URL list.
- **Some are `executeOnce: true`:** indicates expectation of single item (avoid duplicates).
- **Edge cases:** missing images; row not found.

#### Fetching Image URLs / 1 / 2 / 3 (Code)
- **Type/role:** `code` — extracts image URLs list into iterable items.
- **Edge cases:** stored as JSON string vs array; empty list.

#### Getting Image Data / Getting Image Data1 (HTTP Request)
- **Type/role:** `httpRequest` — downloads image (or metadata) from URL for the AI agent context.
- **Failure modes:** 403 hotlink protection; large images; non-image content-type.

#### Image Title & Alt Tag Creation / Image Title & Alt Tag Creation1
- **Type/role:** `langchain.agent` — uses Gemini to produce structured metadata per image.
- **Bindings:**
  - `ai_languageModel` from Gemini Chat Model7/8
  - `ai_outputParser` from Structured Output Parser / 1
- **Edge cases:** agent returns invalid schema; unsafe content; ambiguous images.

#### Structured Output Parser / Structured Output Parser1
- **Type/role:** `outputParserStructured` — enforces schema (e.g., `{ title: string, alt: string }`).
- **Config:** Schema not visible; must be defined to match downstream mapping code.
- **Failure modes:** parse errors; missing keys.

#### Mapping Image URL with Alt Tag / Mapping Image URL with Alt Tag1 (Code)
- **Type/role:** `code` — merges `{url}` with `{alt/title}` into Shopify image objects.
- **Edge cases:** mismatch lengths; duplicates; null alt.

#### Fetching Binary Data / Fetching Binary Data1 (HTTP Request)
- **Type/role:** downloads binary image bytes for Shopify upload payload.
- **Config:** must enable “Download” / “Response: File” producing binary data.
- **Failure modes:** binary not returned; timeouts; large payload.

#### Formatting Images as Base64 / Formatting Images as Base64_1 (Code)
- **Type/role:** converts binary to base64 strings and builds Shopify images array.
- **Edge cases:** base64 size limits; memory usage with many images.

---

### 1.6 Shopify Product Creation & Notifications

**Overview:** Creates the product in Shopify (two mirrored nodes), then notifies user of success.

**Nodes involved:**
- Creating Product on Shopify
- Notifying user about Product Creation
- Creating Product on Shopify1
- Notifying user about Product Creation1

#### Creating Product on Shopify / Creating Product on Shopify1
- **Type/role:** `httpRequest` — calls Shopify Admin API to create product.
- **Config (expected):**
  - Method: POST
  - URL: `https://{shop}.myshopify.com/admin/api/{version}/products.json` (REST) or GraphQL endpoint
  - Auth: Admin API access token (Header `X-Shopify-Access-Token`)
  - Body: product title, body_html/description, handle/slug, variants (prices), images (base64 attachments), alt text
- **Edge cases:** Shopify API versioning; image attachment formatting; 422 validation errors; rate limits (429).

#### Notifying user about Product Creation / 1
- **Type/role:** `telegram` — sends completion message (optionally includes product URL/id).
- **Edge cases:** message formatting; missing product id in response.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Receive Telegram updates | — | Get User State |  |
| Get User State | dataTable | Load user state (step) | Telegram Trigger | If /abort |  |
| If /abort | if | Abort command handling | Get User State | Reset Shopify Product Manager DB; Fetch Data from User State DB |  |
| Reset Shopify Product Manager DB | dataTable | Clear product state | If /abort | Reset User_images DB |  |
| Reset User_images DB | dataTable | Clear image state | Reset Shopify Product Manager DB | Notifying user about process abortion |  |
| Notifying user about process abortion | telegram | Notify abort | Reset User_images DB | — |  |
| Fetch Data from User State DB | code | Normalize state for routing | If /abort | If3 |  |
| If3 | if | Decide whether user has state | Fetch Data from User State DB | Get User State1; Check for /start |  |
| Get User State1 | dataTable | Load state for known users | If3 | Switch |  |
| Check for /start | if | Gate to start flow | If3 | Prompting user for product name; Prompting user to use /Start |  |
| Prompting user for product name | telegram | Ask for product name | Check for /start | Chat ID Update |  |
| Chat ID Update | dataTable | Store chat id / initialize step | Prompting user for product name | — |  |
| Prompting user to use /Start | telegram | Instruct user to send /start | Check for /start | — |  |
| Switch | switch | Route by current step | Get User State1 | Check Approval Button; Updating Current Step & Product Name; Check for Done after uploading images ; Check for None in Features; Updating Current Step2; If5; Prompting user to create another product |  |
| Check Approval Button | if | Validate approval action | Switch | Notifying user about Product Creation Initiation; Upsert row(s)1 |  |
| Upsert row(s)1 | dataTable | Update/reset row on invalid approval | Check Approval Button | Notifying User to Restart |  |
| Notifying User to Restart | telegram | Ask user to restart flow | Upsert row(s)1 | — |  |
| Notifying user about Product Creation Initiation | telegram | Tell user creation is starting | Check Approval Button | Updating Current Step |  |
| Updating Current Step | dataTable | Move to generation step | Notifying user about Product Creation Initiation | Fetching Product Data4 |  |
| Fetching Product Data4 | dataTable | Fetch product data for generation | Updating Current Step | Code in JavaScript |  |
| Code in JavaScript | code | Prepare fields for downstream | Fetching Product Data4 | Edit Fields |  |
| Edit Fields | set | Map/rename fields | Code in JavaScript | Code in JavaScript2 |  |
| Code in JavaScript2 | code | Prepare HTTP inputs | Edit Fields | HTTP Request |  |
| HTTP Request | httpRequest | External fetch/prep before AI branch | Code in JavaScript2 | If2 |  |
| If2 | if | Choose AI generation pipeline | HTTP Request | Product Description Generation; Product Description Generation1 |  |
| Product Description Generation | chainLlm | Generate long description (A) | If2 | Markdown |  |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM for long desc (A) | — | Product Description Generation (ai_languageModel) |  |
| Markdown | markdown | Format description (A) | Product Description Generation | Short Description Generation |  |
| Short Description Generation | chainLlm | Generate short description (A) | Markdown | Slug Generation |  |
| Google Gemini Chat Model1 | lmChatGoogleGemini | LLM for short desc (A) | — | Short Description Generation (ai_languageModel) |  |
| Slug Generation | chainLlm | Generate slug/handle (A) | Short Description Generation | Fetching Product Data2 |  |
| Google Gemini Chat Model3 | lmChatGoogleGemini | LLM for slug (A) | — | Slug Generation (ai_languageModel) |  |
| Product Description Generation1 | chainLlm | Generate long description (B) | If2 | Markdown1 |  |
| Google Gemini Chat Model2 | lmChatGoogleGemini | LLM for long desc (B) | — | Product Description Generation1 (ai_languageModel) |  |
| Markdown1 | markdown | Format description (B) | Product Description Generation1 | Short Description Generation1 |  |
| Short Description Generation1 | chainLlm | Generate short description (B) | Markdown1 | Slug Generation1 |  |
| Google Gemini Chat Model4 | lmChatGoogleGemini | LLM for short desc (B) | — | Short Description Generation1 (ai_languageModel) |  |
| Slug Generation1 | chainLlm | Generate slug/handle (B) | Short Description Generation1 | Fetching Product Data |  |
| Google Gemini Chat Model5 | lmChatGoogleGemini | LLM for slug (B) | — | Slug Generation1 (ai_languageModel) |  |
| Fetching Product Data2 | dataTable | Load images list (A) | Slug Generation | Fetching Image URLs2 |  |
| Fetching Image URLs2 | code | Extract image URLs (A) | Fetching Product Data2 | Getting Image Data1 |  |
| Getting Image Data1 | httpRequest | Download image for AI (A) | Fetching Image URLs2 | Image Title & Alt Tag Creation1 |  |
| Image Title & Alt Tag Creation1 | agent | Generate image title/alt (A) | Getting Image Data1 | Mapping Image URL with Alt Tag1 |  |
| Google Gemini Chat Model8 | lmChatGoogleGemini | LLM for image agent (A) | — | Image Title & Alt Tag Creation1 (ai_languageModel) |  |
| Structured Output Parser1 | outputParserStructured | Enforce schema (A) | — | Image Title & Alt Tag Creation1 (ai_outputParser) |  |
| Mapping Image URL with Alt Tag1 | code | Merge URL + alt/title (A) | Image Title & Alt Tag Creation1 | Fetching Product Data3 |  |
| Fetching Product Data3 | dataTable | Load for binary fetch (A) | Mapping Image URL with Alt Tag1 | Fetching Image URLs3 |  |
| Fetching Image URLs3 | code | Iterate URLs (A) | Fetching Product Data3 | Fetching Binary Data1 |  |
| Fetching Binary Data1 | httpRequest | Download binary (A) | Fetching Image URLs3 | Formatting Images as Base64_1 |  |
| Formatting Images as Base64_1 | code | Base64 encode for Shopify (A) | Fetching Binary Data1 | Creating Product on Shopify1 |  |
| Creating Product on Shopify1 | httpRequest | Create Shopify product (A) | Formatting Images as Base64_1 | Notifying user about Product Creation1 |  |
| Notifying user about Product Creation1 | telegram | Notify success (A) | Creating Product on Shopify1 | — |  |
| Fetching Product Data | dataTable | Load images list (B) | Slug Generation1 | Fetching Image URLs |  |
| Fetching Image URLs | code | Extract image URLs (B) | Fetching Product Data | Getting Image Data |  |
| Getting Image Data | httpRequest | Download image for AI (B) | Fetching Image URLs | Image Title & Alt Tag Creation |  |
| Image Title & Alt Tag Creation | agent | Generate image title/alt (B) | Getting Image Data | Mapping Image URL with Alt Tag |  |
| Google Gemini Chat Model7 | lmChatGoogleGemini | LLM for image agent (B) | — | Image Title & Alt Tag Creation (ai_languageModel) |  |
| Structured Output Parser | outputParserStructured | Enforce schema (B) | — | Image Title & Alt Tag Creation (ai_outputParser) |  |
| Mapping Image URL with Alt Tag | code | Merge URL + alt/title (B) | Image Title & Alt Tag Creation | Fetching Product Data1 |  |
| Fetching Product Data1 | dataTable | Load for binary fetch (B) | Mapping Image URL with Alt Tag | Fetching Image URLs1 |  |
| Fetching Image URLs1 | code | Iterate URLs (B) | Fetching Product Data1 | Fetching Binary Data |  |
| Fetching Binary Data | httpRequest | Download binary (B) | Fetching Image URLs1 | Formatting Images as Base64 |  |
| Formatting Images as Base64 | code | Base64 encode for Shopify (B) | Fetching Binary Data | Creating Product on Shopify |  |
| Creating Product on Shopify | httpRequest | Create Shopify product (B) | Formatting Images as Base64 | Notifying user about Product Creation |  |
| Notifying user about Product Creation | telegram | Notify success (B) | Creating Product on Shopify | — |  |
| Updating Current Step & Product Name | dataTable | Save name and step | Switch | Notifying User to Specify Product Name |  |
| Notifying User to Specify Product Name | telegram | Ask user next action after name | Updating Current Step & Product Name | — |  |
| Check for Done after uploading images  | if | Detect “done” vs more uploads | Switch | Fetching Product Data5; Code |  |
| Code | code | Extract file_id etc. | Check for Done after uploading images  | Get a file |  |
| Get a file | telegram | Telegram getFile API | Code | Edit Fields1 |  |
| Edit Fields1 | set | Normalize image fields | Get a file | Updating Image Data |  |
| Updating Image Data | dataTable | Store image info | Edit Fields1 | — |  |
| Fetching Product Data5 | dataTable | Load uploaded images | Check for Done after uploading images  | Combining File IDs & URLs |  |
| Combining File IDs & URLs | code | Consolidate file ids/urls | Fetching Product Data5 | Updating Current Step  |  |
| Updating Current Step  | dataTable | Advance to features step | Combining File IDs & URLs | Prompting user for Features |  |
| Prompting user for Features | telegram | Ask for features list | Updating Current Step  | — |  |
| Check for None in Features | if | Skip features if none | Switch | Updating Current Step1; Updating Current Step & Features |  |
| Updating Current Step1 | dataTable | Advance to regular price | Check for None in Features | Prompting user for Regular Price |  |
| Prompting user for Regular Price | telegram | Ask for regular price | Updating Current Step1 | — |  |
| Updating Current Step & Features | dataTable | Save features + step | Check for None in Features | Prompting user for Regular Price1 |  |
| Prompting user for Regular Price1 | telegram | Ask for regular price | Updating Current Step & Features | — |  |
| Updating Current Step2 | dataTable | Advance to sale price | Switch | Prompting user for Sale Price |  |
| Prompting user for Sale Price | telegram | Ask for sale price | Updating Current Step2 | — |  |
| If5 | if | Validate/branch sale price logic | Switch | Updating Sale Price; Updating Sale Price1 |  |
| Updating Sale Price | dataTable | Save sale price path A | If5 | Merge |  |
| Updating Sale Price1 | dataTable | Save sale price path B | If5 | Merge |  |
| Merge | merge | Join sale price branches | Updating Sale Price; Updating Sale Price1 | Get All Data |  |
| Get All Data | dataTable | Retrieve final payload data | Merge | Code in JavaScript4 |  |
| Code in JavaScript4 | code | Build preview/approval payload | Get All Data | If6 |  |
| If6 | if | Decide to post group media | Code in JavaScript4 | Telegram Group Media |  |
| Telegram Group Media | httpRequest | Send media preview to group | If6 | Prompting user for Approval |  |
| Prompting user for Approval | telegram | Ask user to approve creation | Telegram Group Media | — |  |
| Prompting user to create another product | telegram | Ask to restart new product | Switch | Reset Shopify Product Manager DB1 |  |
| Reset Shopify Product Manager DB1 | dataTable | Clear state after completion | Prompting user to create another product | Reset User_Images DB |  |
| Reset User_Images DB | dataTable | Clear images after completion | Reset Shopify Product Manager DB1 | — |  |
| Sticky Note … (all) | stickyNote | Comments | — | — | *(empty content in this workflow)* |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Data Tables (n8n “Data Table” feature)**
   1. Create **User State** table with a unique key like `telegramUserId`. Suggested columns: `telegramUserId`, `chatId`, `currentStep`, `productId` (draft), timestamps.
   2. Create **Shopify Product Manager** table with `telegramUserId` key and columns: `productName`, `features`, `regularPrice`, `salePrice`, `slug`, `shortDescription`, `longDescription`, `imageUrls` (array/json), etc.
   3. Create **User_Images** table keyed by `telegramUserId` + image index (or separate row per image) storing: `file_id`, `file_path`, `url`, `title`, `alt`.

2) **Telegram entry**
   1. Add **Telegram Trigger** node; connect Telegram bot credentials.
   2. Configure to receive at least: messages, photos/documents, and callback queries (for approval buttons).

3) **Load user state**
   1. Add **Data Table → Get User State**: query by `telegramUserId` from the Telegram payload.
   2. Add **IF → If /abort** checking message text equals `/abort`.
   3. True branch: add Data Table nodes to reset/clear per-user rows in Shopify Product Manager + User_Images; then add Telegram node to notify abortion.
   4. False branch: add **Code → Fetch Data from User State DB** to output a clean `currentStep` variable.

4) **Gate to /start**
   1. Add **IF → If3**: if state exists/has valid `currentStep`.
   2. False branch: add **IF → Check for /start** (message text).
      - True: Telegram “Prompting user for product name” then Data Table “Chat ID Update” to store chat id and set `currentStep = awaiting_name`.
      - False: Telegram “Prompting user to use /Start”.

5) **Main router**
   1. True branch of **If3**: Data Table **Get User State1** then **Switch** on `currentStep`.
   2. Create switch cases for each step (names are up to you, but must match what you store).

6) **Step: product name**
   1. Switch output for “awaiting_name” → Data Table “Updating Current Step & Product Name” (save product name, set step to `awaiting_images`).
   2. Then Telegram “Notifying User to Specify Product Name” (or next instructions like “upload images, type done when finished”).

7) **Step: images upload**
   1. Switch output for “awaiting_images” → IF “Check for Done after uploading images”.
      - If text is “done”: fetch accumulated images from DB (Fetching Product Data5), Code “Combining File IDs & URLs”, Data Table “Updating Current Step ” set to `awaiting_features`, Telegram “Prompting user for Features”.
      - Else: Code node extracts `file_id` from Telegram update; Telegram node “Get a file” (getFile); Set node “Edit Fields1”; Data Table “Updating Image Data” (append/store).

8) **Step: features**
   1. Switch output “awaiting_features” → IF “Check for None in Features”.
      - If none: Data Table “Updating Current Step1” to `awaiting_regular_price`, Telegram “Prompting user for Regular Price”.
      - Else: Data Table “Updating Current Step & Features”, Telegram “Prompting user for Regular Price1”.

9) **Step: regular price → sale price**
   1. On regular price step: Data Table “Updating Current Step2” to `awaiting_sale_price`, Telegram “Prompting user for Sale Price”.
   2. On sale price step: IF “If5” validating sale price (allow empty/0).
      - Two Data Table updates (Updating Sale Price / Updating Sale Price1), then **Merge**.

10) **Prepare approval preview**
   1. After Merge: Data Table “Get All Data” to load final dataset.
   2. Code “Code in JavaScript4” to format preview.
   3. IF “If6” to decide whether to post preview to a Telegram group/channel.
   4. HTTP Request “Telegram Group Media” (or Telegram node sendMediaGroup) to post preview.
   5. Telegram “Prompting user for Approval” with inline keyboard buttons (Approve/Reject). Store callback data format consistently.

11) **Approval handling and AI generation**
   1. Switch case for “awaiting_approval” → IF “Check Approval Button” checks callback data.
      - If not approved: Upsert row(s)1 + notify restart.
      - If approved: Telegram “Notifying user about Product Creation Initiation”, Data Table “Updating Current Step” to `generating`.
   2. Fetch product data (Fetching Product Data4) and run Code/Set nodes to build prompts.

12) **Gemini setup**
   1. Add **Google Gemini Chat Model** nodes (one per chain/agent) with Gemini API credentials.
   2. Add **chainLlm** nodes for: long description, short description, slug.
   3. Add **Markdown** nodes if you want formatting normalization.
   4. Add **IF (If2)** to choose between pipeline A/B (define the condition explicitly).

13) **Image AI metadata**
   1. For each pipeline, add Data Table fetch → Code to extract URLs → HTTP Request to fetch image → LangChain **Agent**.
   2. Add **Structured Output Parser** with a schema like:
      - `title: string`
      - `alt: string`
   3. Map `{url, alt, title}` with Code nodes.

14) **Download binaries + base64**
   1. HTTP Request to download each image as binary (Response = File).
   2. Code node converts to base64 and builds Shopify payload images array.

15) **Create product in Shopify**
   1. Add HTTP Request “Creating Product on Shopify”:
      - Auth header `X-Shopify-Access-Token: {{TOKEN}}`
      - Endpoint per your API choice (REST/GraphQL).
      - Body includes title, descriptions, prices, images with base64 attachments + alt.
   2. Telegram notify node with created product info.

16) **Reset for new product**
   1. Telegram “Prompting user to create another product”.
   2. Reset/clear per-user rows in Shopify Product Manager and User_Images (Reset Shopify Product Manager DB1, Reset User_Images DB).

**Credentials to configure**
- **Telegram:** Bot token in n8n Telegram credentials.
- **Google Gemini:** Gemini API key/credentials in n8n’s Google Gemini integration.
- **Shopify:** Admin API access token (custom app) stored securely; shop domain; API version.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in this workflow JSON. | No additional embedded documentation provided. |
| Many nodes have empty `parameters` in the provided JSON. | To reproduce precisely, you must re-enter prompts, Data Table queries, Shopify endpoints, Telegram message templates, and IF/Switch conditions. |

If you want, I can infer a **recommended Data Table schema**, a **canonical `currentStep` enumeration**, and **example prompt templates** for the Gemini chain/agent nodes based on the node names and connections.