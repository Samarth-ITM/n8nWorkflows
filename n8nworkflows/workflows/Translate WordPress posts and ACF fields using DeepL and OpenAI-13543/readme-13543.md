Translate WordPress posts and ACF fields using DeepL and OpenAI

https://n8nworkflows.xyz/workflows/translate-wordpress-posts-and-acf-fields-using-deepl-and-openai-13543


# Translate WordPress posts and ACF fields using DeepL and OpenAI

## 1. Workflow Overview

**Purpose:**  
This workflow receives a WordPress post event (via webhook), fetches the full post data from WordPress, detects the source language using OpenAI, determines which target translations are missing, optionally corrects the original post’s language flag in WordPress, then translates Gutenberg content + selected ACF text fields using DeepL, reconstructs the translated post payload while preserving ACF structure, and finally creates translated draft posts in WordPress linked to the original.

**Typical use cases:**
- Multilingual WordPress sites (e.g., Polylang/WPML-like setups) where posts must be replicated per language.
- Sites using **Gutenberg** content and **Advanced Custom Fields (ACF)**, including nested structures (repeaters/flexible content).
- Avoiding duplicate translations by checking existing `translations` metadata from WordPress.

### 1.1 Trigger & Fetch Raw WP Post Data
Receives a POST webhook containing a post identifier/type, then fetches the canonical WP REST API post object.

### 1.2 AI Language Detection & Smart Routing
Builds a short text snippet, detects language via OpenAI, decides whether the WP language flag must be fixed, and computes which target languages still need translations.

### 1.3 Flatten ACF Structure & Translate with DeepL
Extracts translatable strings from post title/content + ACF (based on configured keys), splits execution by each target language, and sends a batch translation request to DeepL (HTML-aware).

### 1.4 Rebuild ACF Object & Push to WordPress
Reapplies translated strings into the correct title/content/ACF paths, cleans the ACF JSON, and creates translated WordPress posts as drafts linked to the original.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Trigger & Fetch Raw WP Post Data

**Overview:**  
This block starts the workflow via webhook and retrieves the complete post JSON from the WordPress REST API using Basic Auth.

**Nodes Involved:**
- Webhook
- Get Post Data

#### Node: Webhook
- **Type / Role:** `n8n-nodes-base.webhook` — Entry point (HTTP trigger).
- **Config (interpreted):**
  - Method: `POST`
  - Path: `translate-post` (endpoint becomes `.../webhook/translate-post` depending on n8n setup)
- **Inputs/Outputs:**
  - **Output →** `Get Post Data`
- **Key data expected in input:**
  - Uses expressions later like: `$json.body.post.ID` and `$json.body.post.post_type`
- **Edge cases / failures:**
  - Missing/invalid payload structure (no `body.post.ID` or `body.post.post_type`) will break URL construction downstream.
  - If webhook is publicly accessible, consider authentication/secret header validation (not implemented here).
- **Version notes:** Node typeVersion `1`.

#### Node: Get Post Data
- **Type / Role:** `n8n-nodes-base.httpRequest` — Fetch post from WP REST API.
- **Config (interpreted):**
  - URL is dynamic:
    - Base: `https://your-wordpress-site.com/wp-json/wp/v2/`
    - Endpoint chosen by `post_type`:
      - if `post` → `posts`
      - if `solution` → `solutions`
      - else → uses the raw `post_type` directly
    - Then appends `/{ID}`
  - Authentication: **HTTP Basic Auth** via n8n “generic credential type”
- **Key expressions:**
  - `{{$json.body.post.post_type}}`, `{{$json.body.post.ID}}`
- **Inputs/Outputs:**
  - **Input ←** `Webhook`
  - **Output →** `Prepare Text Snippet`
- **Edge cases / failures:**
  - 401/403 if Application Password / user lacks permissions.
  - 404 if post type endpoint does not exist or ID invalid.
  - If WP returns a different schema (missing `content.rendered`, `_links.self[0].href`, etc.), later nodes may fail.
- **Sticky notes applying:**
  - “## 1. Trigger & Fetch Raw WP Post Data”
  - “### URL and WP APP Password\nUpdate the URL to your WP domain and select your WP App Password credentials.”
- **Version notes:** typeVersion `3`.

---

### Block 1.2 — AI Language Detection & Smart Routing

**Overview:**  
Creates a short plain-text snippet, detects its language using OpenAI, determines required target translations, and optionally corrects the original post’s language flag in WordPress. A merge node reunifies the “fix language” and “no fix” paths.

**Nodes Involved:**
- Prepare Text Snippet
- OpenAI Language Detect
- Smart Router & Targets
- Needs WP Lang Fix?
- Fix WP Language Flag
- Merge
- Needs Translations?

#### Node: Prepare Text Snippet
- **Type / Role:** `n8n-nodes-base.function` — Build a short sample text for detection.
- **Config (interpreted):**
  - Creates `textSnippet` from:
    - `post.title.rendered`
    - plus stripped HTML from `post.content.rendered` if present
    - else falls back to `post.acf.intro` if present
  - Truncates to 1000 chars to reduce OpenAI token usage.
- **Output:**
  - Passes through the full post JSON plus `textSnippet`.
- **Inputs/Outputs:**
  - **Input ←** `Get Post Data`
  - **Output →** `OpenAI Language Detect`
- **Edge cases / failures:**
  - If `title.rendered` missing, may throw.
  - If content is extremely large, still safe due to substring but HTML stripping can be expensive (usually fine).
- **Version notes:** typeVersion `1`.

#### Node: OpenAI Language Detect
- **Type / Role:** `n8n-nodes-base.openAi` — Chat completion for language detection.
- **Config (interpreted):**
  - Resource: `chat`
  - Model: `gpt-4o-mini`
  - System message instructs: output ONLY 2-letter ISO 639-1 code.
  - Note: The node as shown includes only a system message; in a working setup you typically also include a **user** message containing `{{$json.textSnippet}}`. (If not configured in the UI, detection will not work.)
- **Inputs/Outputs:**
  - **Input ←** `Prepare Text Snippet`
  - **Output →** `Smart Router & Targets`
- **Edge cases / failures:**
  - OpenAI credential/auth errors, rate limits, transient 5xx.
  - Model may output unexpected whitespace/newline; downstream `.trim().toLowerCase()` mitigates.
  - If prompt does not include the snippet, output is undefined/unreliable.
- **Sticky notes applying:**
  - “## 2. AI Language Detection & Smart Routing”
- **Version notes:** typeVersion `1`.

#### Node: Smart Router & Targets
- **Type / Role:** `n8n-nodes-base.function` — Decide detected language, targets, and whether to fix WP language flag.
- **Config (interpreted):**
  - Reads:
    - `originalPost` from `$items("Prepare Text Snippet")[0].json`
    - `detectedLang` from `items[0].json.message.content`
    - `currentWpLang` from `originalPost.lang` (defaults to `en`)
  - Supported languages configured in `allSupported = ['en', 'fr', 'de']` (must be updated to match site).
  - Computes `languagesNeeded`:
    - targets = allSupported excluding detectedLang
    - then removes languages already present in `originalPost.translations` map
  - Computes `needsLangFix` if WP language flag differs from detected language.
  - Outputs `postTypeUrl` from `originalPost._links.self[0].href`
- **Inputs/Outputs:**
  - **Input ←** `OpenAI Language Detect`
  - **Output →** `Needs WP Lang Fix?`
- **Edge cases / failures:**
  - If OpenAI output not ISO code, `toLowerCase()` still works but logic may mis-route.
  - If `_links.self[0].href` missing, language fix and base URL reconstruction later will fail.
  - If `translations` structure differs (not a `{lang: id}` map), duplicate-prevention fails.
- **Sticky notes applying:**
  - “## 2. AI Language Detection & Smart Routing”
  - “### Configuration:\nOpen this node and update the \"allSupported\" array to match your website's languages.”
- **Version notes:** typeVersion `1`.

#### Node: Needs WP Lang Fix?
- **Type / Role:** `n8n-nodes-base.if` — Conditional branch.
- **Condition:**
  - `{{$json.needsLangFix}} == true`
- **Routing:**
  - **True →** `Fix WP Language Flag`
  - **False →** `Merge` (input 2 / index 1)
- **Inputs/Outputs:**
  - **Input ←** `Smart Router & Targets`
- **Edge cases:**
  - If `needsLangFix` is missing/non-boolean, condition may evaluate unexpectedly.
- **Version notes:** typeVersion `1`.

#### Node: Fix WP Language Flag
- **Type / Role:** `n8n-nodes-base.httpRequest` — Updates original post language in WP.
- **Config (interpreted):**
  - URL: `{{$json.postTypeUrl}}` (self link of the original post)
  - Method: `POST`
  - Body parameter: `lang = {{$json.detectedLang}}`
  - Authentication: HTTP Basic Auth (WordPress Application Password)
- **Inputs/Outputs:**
  - **Input ←** `Needs WP Lang Fix?` (true branch)
  - **Output →** `Merge` (input 1 / index 0)
- **Edge cases / failures:**
  - WP may require `PUT` depending on configuration; many WP REST endpoints accept `POST` for updates, but not all.
  - If multilingual plugin uses different field name than `lang`, update will have no effect.
  - Auth/permission errors, or REST API disabled.
- **Version notes:** typeVersion `3`.

#### Node: Merge
- **Type / Role:** `n8n-nodes-base.merge` — Joins the “fixed” and “not fixed” branches back into a single flow.
- **Config:**
  - Default merge behavior (no explicit mode shown).
- **Inputs/Outputs:**
  - **Input 0 ←** `Fix WP Language Flag`
  - **Input 1 ←** `Needs WP Lang Fix?` (false path)
  - **Output →** `Needs Translations?`
- **Edge cases:**
  - Depending on merge mode, you may get unexpected output if both branches run (here only one runs).
- **Version notes:** typeVersion `2.1`.

#### Node: Needs Translations?
- **Type / Role:** `n8n-nodes-base.if` — Stops early if nothing to translate.
- **Condition:**
  - `{{$json.languagesNeeded.length > 0}} == true`
- **Routing:**
  - **True →** `Extract Content`
  - **False →** flow ends (no further nodes)
- **Inputs/Outputs:**
  - **Input ←** `Merge`
- **Edge cases:**
  - If `languagesNeeded` missing/not an array, expression errors.
- **Version notes:** typeVersion `1`.

---

### Block 1.3 — Flatten ACF Structure & Translate with DeepL

**Overview:**  
Builds an ordered list of strings to translate and a matching list of “map paths” indicating where each translated string should be written back. Then splits the execution into one item per target language and sends the translation request to DeepL.

**Nodes Involved:**
- Extract Content
- Split by Target Lang
- DeepL Translate

#### Node: Extract Content
- **Type / Role:** `n8n-nodes-base.code` — Extracts translatable strings from Gutenberg + ACF, preserving mapping.
- **Config (interpreted):**
  - Builds:
    - `stringsToTranslate`: ordered array
    - `mapPaths`: same length; entries like `post_title`, `post_content`, `acf.intro`, `acf.product_flexible[0].featureTitle`, etc.
  - Always includes `post.title.rendered` mapped to `post_title`
  - Includes `post.content.rendered` mapped to `post_content` if non-empty
  - Adds `acf.intro` if present
  - Walks nested ACF objects/arrays to find keys listed in `textKeys` (must be customized to your ACF schema)
  - Special handling: if `acf.product_flexible` exists, it recursively walks it
- **Inputs/Outputs:**
  - **Input ←** `Needs Translations?` (true branch)
  - **Output →** `Split by Target Lang`
- **Edge cases / failures:**
  - If ACF field names differ from `textKeys`, important text won’t be translated.
  - If ACF contains rich structures with unexpected types, walk() is defensive but can still traverse huge objects (performance).
  - Gutenberg content is translated as HTML; later DeepL uses `tag_handling=html`.
- **Sticky notes applying:**
  - “## 3. Flatten ACF Structure & Translate with DeepL”
  - “### ACF Keys\nOpen this node and update the \"textKeys\" array to match the actual ACF field names used on your WordPress site”
- **Version notes:** typeVersion `2` (Code node, JS).

#### Node: Split by Target Lang
- **Type / Role:** `n8n-nodes-base.itemLists` — Creates one item per needed target language.
- **Config (interpreted):**
  - Splits field: `languagesNeeded`
  - Include: “allOtherFields” (keeps the rest of the JSON on each split item)
  - `alwaysOutputData: true` (prevents empty output in some scenarios)
- **Resulting shape:**
  - Each item has `languagesNeeded` set to a single language (string), plus original data.
- **Inputs/Outputs:**
  - **Input ←** `Extract Content`
  - **Output →** `DeepL Translate`
- **Edge cases:**
  - If `languagesNeeded` is empty, behavior depends on node internals; upstream IF prevents this.
- **Version notes:** typeVersion `1`.

#### Node: DeepL Translate
- **Type / Role:** `n8n-nodes-base.httpRequest` — Calls DeepL translate API.
- **Config (interpreted):**
  - URL: `https://api-free.deepl.com/v2/translate` (free endpoint; paid is `api.deepl.com`)
  - Method: `POST`
  - Headers:
    - `Authorization: DeepL-Auth-Key YOUR_DEEPL_API_KEY_HERE` (must be replaced)
  - Body parameters:
    - `text = {{$json.stringsToTranslate}}` (array; DeepL supports repeated `text` parameters—n8n will serialize)
    - `target_lang = {{$json.languagesNeeded.toUpperCase()}}`
    - `source_lang = {{$json.detectedLang.toUpperCase()}}`
    - `tag_handling = html`
- **Inputs/Outputs:**
  - **Input ←** `Split by Target Lang`
  - **Output →** `RECONSTRUCT JSON`
- **Edge cases / failures:**
  - 403 if key invalid/wrong endpoint (free vs paid mismatch).
  - DeepL target language codes differ from ISO639-1 in some cases (e.g., `EN-GB`, `PT-PT`); this workflow assumes simple 2-letter codes.
  - HTML handling can still mangle Gutenberg blocks in edge cases; consider `preserve_formatting` or glossary if needed.
- **Sticky notes applying:**
  - “## 3. Flatten ACF Structure & Translate with DeepL”
  - “### DEEPL Translate\nAdd your DeepL API Key in the Header Parameters.”
- **Version notes:** typeVersion `3`.

---

### Block 1.4 — Rebuild ACF Object & Push to WordPress

**Overview:**  
Matches DeepL’s returned translations to the original `mapPaths`, writes them back into a cloned post object (title/content/ACF), cleans empty strings, then POSTs a new translated draft post to WordPress with translation linkage.

**Nodes Involved:**
- RECONSTRUCT JSON
- Create Translated Post

#### Node: RECONSTRUCT JSON
- **Type / Role:** `n8n-nodes-base.code` — Rebuilds a translated WP payload per language.
- **Config (interpreted):**
  - Pulls the original split items via `$items("Split by Target Lang")` to align translation results with the correct language and map paths.
  - For each translated item:
    - `translatedTexts = items[i].json.translations` (expects DeepL response shape: array of `{ text: "..." }`)
    - Clones `originalPost`
    - For each `mapPaths[index]`:
      - `post_title` → writes `newPost.title.raw` + `newPost.title.rendered`
      - `post_content` → writes `newPost.content.raw` + `newPost.content.rendered`
      - `acf.*` → uses a custom `set()` to write nested values (supports arrays via `[index]`)
    - Runs `cleanAcf()`:
      - converts empty strings `""` to `null` recursively
    - Derives `createUrl`:
      - takes `originalData.postTypeUrl` and removes the trailing `/{id}` segment
  - Outputs per language:
    - `lang` (target language)
    - `originalId`
    - `newTitle`, `newAcf`, `newContent`
    - `createUrl` (collection endpoint)
- **Inputs/Outputs:**
  - **Input ←** `DeepL Translate`
  - **Output →** `Create Translated Post`
- **Edge cases / failures:**
  - Assumes DeepL response is `items[i].json.translations` and index-aligned with `stringsToTranslate`; if DeepL returns errors/partial results, mapping breaks.
  - `set()` assumes intermediate objects/arrays exist; if ACF structure differs between original and expected path, may throw.
  - `splitItems[i]` alignment relies on n8n preserving item order between Split → DeepL → this node; usually true, but consider adding explicit correlation IDs if you later parallelize.
- **Sticky notes applying:**
  - “## 4. Rebuild ACF Object & Push to WordPress”
- **Version notes:** typeVersion `2`.

#### Node: Create Translated Post
- **Type / Role:** `n8n-nodes-base.httpRequest` — Creates translated WP post (draft).
- **Config (interpreted):**
  - URL: `{{$json.createUrl}}` (e.g., `https://.../wp-json/wp/v2/posts`)
  - Method: `POST`
  - Auth: HTTP Basic Auth (WordPress Application Password)
  - Body parameters:
    - `title = {{$json.newTitle}}`
    - `acf = {{$json.newAcf}}`
    - `status = draft`
    - `lang = {{$json.lang}}`
    - `translation_of = {{$json.originalId}}`
    - `content` expression is set as `=={{ $json.newContent }}` (note the double `==`, likely a mistake; should be `={{ $json.newContent }}`)
- **Inputs/Outputs:**
  - **Input ←** `RECONSTRUCT JSON`
  - Output: ends workflow
- **Edge cases / failures:**
  - **Expression bug risk:** `content` uses `=={{ ... }}` which may post literal text or fail; should be corrected.
  - WP may reject `acf` unless ACF-to-REST is enabled and user has permissions.
  - `translation_of` and `lang` fields depend on multilingual plugin; if unsupported, these fields will be ignored or cause validation errors.
- **Sticky notes applying:**
  - “## 4. Rebuild ACF Object & Push to WordPress”
  - “### URL and WP APP Password\nUpdate the URL to your WP domain and select your WP App Password credentials.”
- **Version notes:** typeVersion `3`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook Trigger | Receives WP post event payload | — | Get Post Data | ## 1. Trigger & Fetch Raw WP Post Data |
| Get Post Data | HTTP Request | Fetches full post JSON from WP REST API | Webhook | Prepare Text Snippet | ## 1. Trigger & Fetch Raw WP Post Data; ### URL and WP APP Password\nUpdate the URL to your WP domain and select your WP App Password credentials. |
| Prepare Text Snippet | Function | Builds 1000-char snippet for language detection | Get Post Data | OpenAI Language Detect | ## 2. AI Language Detection & Smart Routing |
| OpenAI Language Detect | OpenAI (Chat) | Detects source language ISO code | Prepare Text Snippet | Smart Router & Targets | ## 2. AI Language Detection & Smart Routing |
| Smart Router & Targets | Function | Computes needed target languages + whether to fix WP lang | OpenAI Language Detect | Needs WP Lang Fix? | ## 2. AI Language Detection & Smart Routing; ### Configuration:\nOpen this node and update the "allSupported" array to match your website's languages. |
| Needs WP Lang Fix? | IF | Branch: update WP language flag or skip | Smart Router & Targets | Fix WP Language Flag; Merge | ## 2. AI Language Detection & Smart Routing |
| Fix WP Language Flag | HTTP Request | Updates original post `lang` in WP | Needs WP Lang Fix? | Merge | ## 2. AI Language Detection & Smart Routing |
| Merge | Merge | Rejoins “fixed” and “not fixed” paths | Fix WP Language Flag; Needs WP Lang Fix? | Needs Translations? | ## 2. AI Language Detection & Smart Routing |
| Needs Translations? | IF | Stops if no missing translations | Merge | Extract Content | ## 3. Flatten ACF Structure & Translate with DeepL |
| Extract Content | Code | Flattens title/content/ACF strings + creates mapping paths | Needs Translations? | Split by Target Lang | ## 3. Flatten ACF Structure & Translate with DeepL; ### ACF Keys\nOpen this node and update the "textKeys" array to match the actual ACF field names used on your WordPress site |
| Split by Target Lang | Item Lists | One item per language to translate | Extract Content | DeepL Translate | ## 3. Flatten ACF Structure & Translate with DeepL |
| DeepL Translate | HTTP Request | Batch-translates strings with HTML handling | Split by Target Lang | RECONSTRUCT JSON | ## 3. Flatten ACF Structure & Translate with DeepL; ### DEEPL Translate\nAdd your DeepL API Key in the Header Parameters. |
| RECONSTRUCT JSON | Code | Rebuilds translated post payload + ACF structure | DeepL Translate | Create Translated Post | ## 4. Rebuild ACF Object & Push to WordPress |
| Create Translated Post | HTTP Request | Creates translated draft post linked to original | RECONSTRUCT JSON | — | ## 4. Rebuild ACF Object & Push to WordPress; ### URL and WP APP Password\nUpdate the URL to your WP domain and select your WP App Password credentials. |
| Sticky Note | Sticky Note | Project info / prerequisites / setup notes | — | — | # Smart WP Translation (Gutenberg + ACF)\n\nThis workflow automatically translates new or updated WordPress posts into multiple languages using DeepL, while keeping your Advanced Custom Fields (ACF) JSON structure intact. It uses OpenAI to detect the source language and prevent duplicate translations.\n\n🛠️ Prerequisites:\n1. WordPress Application Password (for HTTP Basic Auth)\n2. OpenAI API Key\n3. DeepL API Key\n\n🚀 Setup Instructions:\n1. Add your credentials to the Webhook, Get Post Data, OpenAI, and Create Post nodes.\n2. In the "Smart Router & Targets" node, configure your supported languages.\n3. In the "Extract Content" node, update the array with your site's specific ACF keys.\n4. In the "DeepL Translate" node, add your DeepL API Key to the Header.\n\nP.S.\n\nYou can use the Deep Node, but in this template I use a generic node in case you get issues with the Deepl Node |
| Sticky Note1 | Sticky Note | Section label | — | — |  |
| Sticky Note2 | Sticky Note | Section label | — | — |  |
| Sticky Note3 | Sticky Note | Section label | — | — |  |
| Sticky Note4 | Sticky Note | Section label | — | — |  |
| Sticky Note5 | Sticky Note | Reminder | — | — |  |
| Sticky Note6 | Sticky Note | Reminder | — | — |  |
| Sticky Note7 | Sticky Note | Reminder | — | — |  |
| Sticky Note8 | Sticky Note | Reminder | — | — |  |
| Sticky Note9 | Sticky Note | Reminder | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Webhook trigger**
   1. Add node: **Webhook**
   2. Set:
      - HTTP Method: `POST`
      - Path: `translate-post`
   3. (Recommended) Add authentication/secret validation (not included in template).

2. **Fetch the WordPress post from REST API**
   1. Add node: **HTTP Request** named **Get Post Data**
   2. Configure:
      - Authentication: **HTTP Basic Auth** (use WP Application Password credentials)
      - URL (expression):  
        `https://your-wordpress-site.com/wp-json/wp/v2/{{ $json.body.post.post_type === 'post' ? 'posts' : $json.body.post.post_type === 'solution' ? 'solutions' : $json.body.post.post_type }}/{{ $json.body.post.ID }}`
   3. Create credentials in n8n:
      - Credential type: HTTP Basic Auth
      - Username: WP username
      - Password: WordPress Application Password
   4. Connect: **Webhook → Get Post Data**

3. **Prepare a short text snippet**
   1. Add node: **Function** named **Prepare Text Snippet**
   2. Paste logic that:
      - Builds `textSnippet` from `title.rendered` and stripped `content.rendered` (or `acf.intro`)
      - Truncates to 1000 chars
   3. Connect: **Get Post Data → Prepare Text Snippet**

4. **Detect language with OpenAI**
   1. Add node: **OpenAI** named **OpenAI Language Detect**
   2. Resource: **Chat**
   3. Model: `gpt-4o-mini`
   4. Messages:
      - System: instruction to output only 2-letter code
      - User: include the snippet, e.g. `{{ $json.textSnippet }}`
   5. Add OpenAI credentials (API key).
   6. Connect: **Prepare Text Snippet → OpenAI Language Detect**

5. **Compute routing (targets + fix flag)**
   1. Add node: **Function** named **Smart Router & Targets**
   2. Implement:
      - `allSupported` array (edit to match your site, e.g. `['en','fr','de']`)
      - `detectedLang` from OpenAI output
      - Remove already existing translations using `originalPost.translations`
      - Output: `languagesNeeded`, `needsLangFix`, `postTypeUrl`, `originalId`
   3. Connect: **OpenAI Language Detect → Smart Router & Targets**

6. **Optional: Fix original WP language flag**
   1. Add node: **IF** named **Needs WP Lang Fix?**
      - Condition: `{{ $json.needsLangFix }} is true`
   2. Add node: **HTTP Request** named **Fix WP Language Flag**
      - URL: `{{ $json.postTypeUrl }}`
      - Method: `POST` (or adapt to PUT if your WP endpoint requires it)
      - Auth: same WP Basic Auth credential
      - Body: `lang = {{ $json.detectedLang }}`
   3. Add node: **Merge** named **Merge** to rejoin branches
      - Connect:
        - IF true → Fix WP Language Flag → Merge (Input 1)
        - IF false → Merge (Input 2)

7. **Stop if nothing to translate**
   1. Add node: **IF** named **Needs Translations?**
      - Condition: `{{ $json.languagesNeeded.length > 0 }} is true`
   2. Connect: **Merge → Needs Translations?**

8. **Extract/flatten content and ACF strings**
   1. Add node: **Code** named **Extract Content**
   2. Configure:
      - Maintain `textKeys` list and **customize it** to your ACF field names.
      - If you have flexible/repeater fields, adapt the `walk()` entry points (e.g., `acf.product_flexible`).
   3. Connect: **Needs Translations? (true) → Extract Content**

9. **Split per target language**
   1. Add node: **Item Lists** named **Split by Target Lang**
   2. Operation: split out items by field `languagesNeeded`
   3. Include: “all other fields”
   4. Connect: **Extract Content → Split by Target Lang**

10. **Translate with DeepL**
   1. Add node: **HTTP Request** named **DeepL Translate**
   2. Configure:
      - URL: `https://api-free.deepl.com/v2/translate` (or paid endpoint)
      - Method: `POST`
      - Header: `Authorization = DeepL-Auth-Key <YOUR_KEY>`
      - Body:
        - `text = {{ $json.stringsToTranslate }}`
        - `source_lang = {{ $json.detectedLang.toUpperCase() }}`
        - `target_lang = {{ $json.languagesNeeded.toUpperCase() }}`
        - `tag_handling = html`
   3. Connect: **Split by Target Lang → DeepL Translate**

11. **Reconstruct translated payload**
   1. Add node: **Code** named **RECONSTRUCT JSON**
   2. Implement:
      - Reapply DeepL translations to the cloned post using `mapPaths`
      - Write back to title/content/ACF
      - Clean empty strings in ACF (optional but recommended)
      - Compute `createUrl` from the original self URL
   3. Connect: **DeepL Translate → RECONSTRUCT JSON**

12. **Create translated draft in WordPress**
   1. Add node: **HTTP Request** named **Create Translated Post**
   2. Configure:
      - URL: `{{ $json.createUrl }}`
      - Method: `POST`
      - Auth: WP Basic Auth credential
      - Body:
        - `title = {{ $json.newTitle }}`
        - `content = {{ $json.newContent }}` (**ensure the expression starts with a single `=`**)
        - `acf = {{ $json.newAcf }}`
        - `status = draft`
        - `lang = {{ $json.lang }}`
        - `translation_of = {{ $json.originalId }}` (plugin-dependent)
   3. Connect: **RECONSTRUCT JSON → Create Translated Post**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Smart WP Translation (Gutenberg + ACF): automatically translates posts into multiple languages using DeepL, preserves ACF JSON, uses OpenAI to detect source language and prevent duplicates. | Workflow sticky note (project overview) |
| Prerequisites: WordPress Application Password (HTTP Basic Auth), OpenAI API key, DeepL API key. | Workflow sticky note (prerequisites) |
| Setup: update WP domain URLs, configure supported languages in “Smart Router & Targets”, update ACF `textKeys` in “Extract Content”, add DeepL key in “DeepL Translate” header. | Workflow sticky note (setup instructions) |
| “You can use the DeepL node, but in this template I use a generic node in case you get issues with the Deepl Node.” | Workflow sticky note (implementation choice) |