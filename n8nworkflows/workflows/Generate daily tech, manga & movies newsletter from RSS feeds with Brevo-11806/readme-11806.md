Generate daily tech, manga & movies newsletter from RSS feeds with Brevo

https://n8nworkflows.xyz/workflows/generate-daily-tech--manga---movies-newsletter-from-rss-feeds-with-brevo-11806


# Generate daily tech, manga & movies newsletter from RSS feeds with Brevo

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Generate daily tech, manga & movies newsletter from RSS feeds with Brevo  
**Internal name:** “RSS par Catégorie vers HTML”

**Purpose:**  
This workflow periodically pulls RSS items from category-based sources stored in Google Sheets, builds a styled HTML newsletter (multi-tab layout), and sends it via **Brevo (Sendinblue)** and/or **Gmail**.

**Target use cases:**
- Personal daily/periodic newsletter for curated topics (manga, cinema, video games).
- Extensible category-to-template pipeline where RSS sources can be managed in a spreadsheet.

### 1.1 Scheduling & Category List
Triggers on a fixed interval, produces a list of categories to process.

### 1.2 Source Lookup (Google Sheets) & Routing
For each category, look up RSS feed URLs in Google Sheets, then route to the correct processing path (Manga / Cinéma / Jeux Vidéo).

### 1.3 RSS Fetching & Item Limiting
For each routed category, fetch RSS items, loop in batches, and keep only the first N items (here: 3).

### 1.4 HTML Snippet Construction (per category)
Convert selected RSS items into HTML article cards (one HTML snippet per category).

### 1.5 Combine, Wrap Template, Send
Merge category snippets, aggregate into a structure referenced by the HTML template, render full newsletter HTML, then deliver via Brevo and Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Category List

**Overview:**  
Starts the workflow on a schedule and defines the set of categories to process.

**Nodes involved:**
- Schedule Trigger
- Category Rss
- Sticky Note - Overview
- Sticky Note1 (Daily trigger)

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — Entry point; runs periodically.
- **Config (interpreted):** Runs every **5 hours** at **minute 30** (`hoursInterval: 5`, `triggerAtMinute: 30`).
- **Outputs:** Emits a single item to start the workflow.
- **Connections:** → `Category Rss`
- **Edge cases / failures:**
- If n8n instance is down at trigger time, the run is skipped unless n8n’s queue/retry strategy is configured externally.
- Timezone depends on n8n instance settings.

#### Node: Category Rss
- **Type / role:** `n8n-nodes-base.set` — Produces a JSON payload with an array of categories.
- **Config (interpreted):** “Raw” JSON output containing:
  - `cat`: list of categories (includes many: Actualités Générales, Technologie, …, Manga).
- **Connections:** ← `Schedule Trigger`; → `Split Out2`
- **Key data produced:** `{"cat":[ ... ]}`
- **Edge cases:**
- Categories must exactly match the values expected downstream (notably the Google Sheets lookup column and the Switch rules). In this workflow, only **Manga**, **Cinéma**, **Jeux Vidéo** are actively routed; other categories will not be processed unless additional Switch rules/paths are added.

#### Node: Sticky Note - Overview (commentary node)
- **Type / role:** `stickyNote` — Documentation on canvas.
- **Content highlights:** Describes daily trigger at 12PM, RSS fetching, HTML templating, and email sending; setup steps and customization instructions.
- **Note:** The actual trigger configured is every 5 hours at minute 30 (not 12PM daily).

#### Node: Sticky Note1
- **Type / role:** `stickyNote` — “## Daily trigger”

---

### Block 2 — Split Categories, Lookup Sources, Route by Category

**Overview:**  
Splits the category array into individual items, queries Google Sheets to find RSS feeds for that category, then routes each feed row to the correct processing pipeline.

**Nodes involved:**
- Split Out2
- Rss database
- Switch
- Sticky Note - Sources1 (disabled)

#### Node: Split Out2
- **Type / role:** `n8n-nodes-base.splitOut` — Converts `cat[]` array into one item per category.
- **Config:** `fieldToSplitOut: "cat"`
- **Input:** From `Category Rss` (one item with `cat` array)
- **Output:** Multiple items, each with one `cat` value.
- **Connections:** → `Rss database`
- **Edge cases:**
- If `cat` is missing or not an array, this node produces no items (workflow effectively stops).

#### Node: Rss database
- **Type / role:** `n8n-nodes-base.googleSheets` — Looks up RSS sources by category.
- **Config (interpreted):**
  - Document: “Rss Database 2” (Google Sheets)
  - Sheet: “Sheet1”
  - Filter: column **Catégorie** equals `{{$json.cat}}`
- **Credentials:** Google Sheets OAuth2
- **Output:** One item per matching spreadsheet row (expected to include at least `Catégorie` and `Lien RSS`).
- **Connections:** ← `Split Out2`; → `Switch`
- **Edge cases / failures:**
- OAuth token expired / permission denied.
- Sheet schema mismatch (column names must match exactly: **Catégorie**, **Lien RSS**).
- No matching rows: `Switch` receives zero items for that category, so nothing is produced downstream.

#### Node: Switch
- **Type / role:** `n8n-nodes-base.switch` — Routes items based on `Catégorie`.
- **Config (interpreted):** 3 rules:
  1. If `{{$json["Catégorie"]}} == "Manga"` → output 0
  2. If `... == "Cinéma"` → output 1
  3. If `... == "Jeux Vidéo"` → output 2
- **Connections:**
  - Output 0 → `Loop Over Items3` (Manga pipeline)
  - Output 1 → `Loop Over Items4` (Movies pipeline)
  - Output 2 → `Loop Over Items5` (Game pipeline)
- **Edge cases:**
- Any category not matching exactly is dropped (no “fallback” path configured).
- Accents/casing matter (“Cinéma” vs “Cinema”).

#### Node: Sticky Note - Sources1 (disabled)
- **Type / role:** `stickyNote` — “Define which RSS feeds to monitor…”
- **Status:** Disabled but still relevant as documentation.

---

### Block 3 — Manga RSS Fetch, Limit Items, Build HTML Snippet

**Overview:**  
Takes RSS feed rows for Manga, fetches RSS items, processes them in batches, keeps the first 3 items, and converts them into an HTML snippet.

**Nodes involved:**
- Loop Over Items3
- RSS Read1
- Code in Python (Beta)
- Manga
- Sticky Note - Processing1 (disabled)

#### Node: Loop Over Items3
- **Type / role:** `n8n-nodes-base.splitInBatches` — Batch-loop controller.
- **Config:** `batchSize: 2`
- **How it works here:**  
  - “Main output 1” goes to the processing branch (`Code in Python (Beta)`), but in this workflow it’s used a bit unusually: it also loops into RSS fetching.
  - “Main output 2” triggers the next batch by looping to `RSS Read1`.
- **Connections:**
  - Output 0 → `Code in Python (Beta)`
  - Output 1 → `RSS Read1`
  - `RSS Read1` → back to `Loop Over Items3` (continuation)
- **Edge cases:**
- If the Google Sheet returns many RSS rows, this node batches them by 2.
- Miswiring risk: SplitInBatches is typically “batch -> do work -> continue”; here, RSS reading is inside the loop chain. It works, but is easy to break if connections are edited.

#### Node: RSS Read1
- **Type / role:** `n8n-nodes-base.rssFeedRead` — Fetch RSS items from URL.
- **Config:** `url: {{$json["Lien RSS"]}}`
- **Error behavior:** `onError: continueErrorOutput`, `alwaysOutputData: true`
  - If a feed fails, workflow continues and still outputs an item (may contain error metadata).
- **Connections:** → `Loop Over Items3` (to continue batching)
- **Edge cases / failures:**
- Invalid RSS URL, network timeouts, rate limits.
- Some RSS feeds omit fields used later (`content`, `creator`), causing downstream Python code to error unless handled.

#### Node: Code in Python (Beta)
- **Type / role:** `n8n-nodes-base.code` (Python) — Limits item count.
- **Config (interpreted):**
  - Reads all incoming items `_input.all()`
  - Returns only first 3: `return _input.all()[:3]`
- **Connections:** → `Manga`
- **Edge cases:**
- If there are fewer than 3 items, returns all available.
- If upstream outputs error-shaped items (from RSS read failure), these may pass through and break HTML generation.

#### Node: Manga
- **Type / role:** `n8n-nodes-base.code` (Python) — Builds HTML snippet for manga items.
- **Logic (interpreted):**
  - For each input item, reads: `title`, `link`, `content`
  - Appends an `<article class="article"> ...` block
  - Returns a single item: `{"HTML": {"html1": "<article...>...</article>..."}}`
- **Connections:** → `Merge` input 0
- **Edge cases:**
- If `content` contains HTML fragments or very long text, it will be injected as-is.
- Missing fields cause runtime errors (`item.json.content` absent). Consider using `.get()` or defaults.

#### Node: Sticky Note - Processing1 (disabled)
- **Type / role:** `stickyNote` — “Split, loop through feeds in batches, and fetch content”.

---

### Block 4 — Movies RSS Fetch, Limit Items, Build HTML Snippet

**Overview:**  
Same as Manga block, but for “Cinéma” category and producing `html2`.

**Nodes involved:**
- Loop Over Items4
- RSS Read2
- Code in Python (Beta)1
- Movie

#### Node: Loop Over Items4
- **Type / role:** `splitInBatches`
- **Config:** batchSize 2
- **Connections:** Output 0 → `Code in Python (Beta)1`, Output 1 → `RSS Read2`, and `RSS Read2` → back to `Loop Over Items4`

#### Node: RSS Read2
- **Type / role:** `rssFeedRead`
- **Config:** `url: {{$json["Lien RSS"]}}`
- **Error behavior:** continue on error, always output data.

#### Node: Code in Python (Beta)1
- **Type / role:** Python code node — Limits to first 3 items.
- **Connections:** → `Movie`

#### Node: Movie
- **Type / role:** Python code node — Builds HTML snippet `{"HTML": {"html2": ...}}`
- **Fields used:** `title`, `link`, `content`, `creator` (assigned but not used in output)
- **Connections:** → `Merge` input 1
- **Edge cases:** same as Manga; also `creator` missing may break unless handled (even if not used, it’s still read).

---

### Block 5 — Video Games RSS Fetch, Limit Items, Build HTML Snippet

**Overview:**  
Same pattern for “Jeux Vidéo” category and producing `html3`.

**Nodes involved:**
- Loop Over Items5
- RSS Read3
- Code in Python (Beta)2
- Jeu video

#### Node: Loop Over Items5
- **Type / role:** `splitInBatches`
- **Config:** batchSize 2
- **Connections:** Output 0 → `Code in Python (Beta)2`, Output 1 → `RSS Read3`, and `RSS Read3` → back to `Loop Over Items5`

#### Node: RSS Read3
- **Type / role:** `rssFeedRead`
- **Config:** `url: {{$json["Lien RSS"]}}`
- **Error behavior:** continue on error, always output data.

#### Node: Code in Python (Beta)2
- **Type / role:** Python code node — Limits to first 3 items.
- **Connections:** → `Jeu video`

#### Node: Jeu video
- **Type / role:** Python code node — Builds HTML snippet `{"HTML": {"html3": ...}}`
- **Fields used:** `title`, `link`, `content`, `creator` (assigned but not used)
- **Connections:** → `Merge` input 2

---

### Block 6 — Merge Snippets, Aggregate Structure, Render Full HTML

**Overview:**  
Combines the three category HTML snippets into a single structure, then renders a full newsletter HTML using an HTML template node that references the aggregated data.

**Nodes involved:**
- Merge
- Aggregate
- HTML Template
- Sticky Note - Combine1 (disabled)
- Sticky Note - Combine2 (disabled)

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — Combines 3 inputs into one output stream.
- **Config:** `numberInputs: 3`
- **Inputs:**
  - Input 0 from `Manga`
  - Input 1 from `Movie`
  - Input 2 from `Jeu video`
- **Output:** One merged item set (n8n “merge by position” semantics depend on configuration; with `numberInputs`, it effectively waits for all branches).
- **Connections:** → `Aggregate`
- **Edge cases:**
- If one branch produces no items (e.g., no RSS rows for that category), the merge may not behave as intended (can stall or output partial results depending on merge mode/version). This workflow assumes all three categories produce output.

#### Node: Aggregate
- **Type / role:** `n8n-nodes-base.aggregate` — Aggregates all incoming items into a single item with an array field.
- **Config (interpreted):**
  - Mode: “aggregateAllItemData”
  - Destination field: `HTML`
- **Output shape:** One item like:
  - `{"HTML": [ {HTML:{html1:...}}, {HTML:{html2:...}}, {HTML:{html3:...}} ] }`
- **Connections:** → `HTML Template`
- **Edge cases:**
- If Merge outputs unexpected shapes, template indexing (`[0]`, `[1]`, `[2]`) may mismatch.

#### Node: HTML Template
- **Type / role:** `n8n-nodes-base.html` — Produces the final HTML body.
- **Config (interpreted):**
  - Large HTML document with CSS + tab sections.
  - Injects snippets via expressions:
    - Anime: `{{ $json.HTML[0].HTML.html1 }}`
    - Movies: `{{ $json.HTML[1].HTML.html2 }}`
    - Game: `{{ $json.HTML[2].HTML.html3 }}`
- **Connections:** → `Mail Campaign` and → `Send a Mail`
- **Important integration note:** Many email clients ignore or restrict `<script>` and complex interactive tabs. The template includes JavaScript tab switching; in most email clients, the tabs will not function as intended.
- **Edge cases:**
- If the aggregate order differs, the wrong snippets appear in the wrong section.
- If any snippet contains unescaped characters, it may break HTML structure.

#### Sticky notes (disabled):
- **Sticky Note - Combine1:** “Set HTML part”
- **Sticky Note - Combine2:** “Merge all feeds and convert to CSV for AI” (note: no CSV/AI nodes exist in this JSON; this comment appears outdated).

---

### Block 7 — Delivery (Brevo + Gmail)

**Overview:**  
Sends the generated HTML via Brevo (Sendinblue) and Gmail.

**Nodes involved:**
- Mail Campaign (Brevo / Sendinblue)
- Send a Mail (Gmail)
- Sticky Note - Email1 (disabled)

#### Node: Mail Campaign
- **Type / role:** `n8n-nodes-base.sendInBlue` — Sends an email campaign/message via Brevo.
- **Config (interpreted):**
  - Sender: `user@example.com`
  - Recipient(s): `user@example.com`
  - Subject: “Newsletter”
  - `sendHTML: true`
  - `htmlContent: {{$json.html}}`
- **Credentials:** Sendinblue/Brevo API credential
- **Input dependency:** Expects an incoming field `html` from `HTML Template`.
- **Edge cases / failures:**
- If `HTML Template` outputs `html` under a different property name, this fails or sends empty body.
- Brevo API key invalid, quota exceeded, sender not authorized.

#### Node: Send a Mail
- **Type / role:** `n8n-nodes-base.gmail` — Sends email via Gmail OAuth2.
- **Config (interpreted):**
  - To: `user@example.com`
  - Subject: “Newsletter”
  - Message body: `{{$json.html}}`
- **Error behavior:** `onError: continueRegularOutput`, `alwaysOutputData: true`
- **Credentials:** Gmail OAuth2
- **Edge cases / failures:**
- Gmail OAuth revoked/expired.
- Gmail may strip scripts/styles; HTML rendering may differ from intended.

#### Sticky Note - Email1 (disabled)
- **Type / role:** `stickyNote` — “Format and send the personalized newsletter.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | stickyNote | Canvas documentation | — | — | # 📬 Personal Daily Tech/Manga/Movies Newsletter; setup/customize steps |
| Schedule Trigger | scheduleTrigger | Periodic execution | — | Category Rss | ## Daily trigger |
| Category Rss | set | Define list of categories | Schedule Trigger | Split Out2 |  |
| Split Out2 | splitOut | Split categories array into items | Category Rss | Rss database |  |
| Rss database | googleSheets | Lookup RSS rows by category | Split Out2 | Switch |  |
| Switch | switch | Route by “Catégorie” | Rss database | Loop Over Items3; Loop Over Items4; Loop Over Items5 |  |
| Loop Over Items3 | splitInBatches | Batch loop controller (Manga path) | Switch; RSS Read1 | Code in Python (Beta); RSS Read1 | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| RSS Read1 | rssFeedRead | Fetch RSS items (Manga) | Loop Over Items3 | Loop Over Items3 | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Code in Python (Beta) | code (python) | Keep first 3 items (Manga) | Loop Over Items3 | Manga | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Manga | code (python) | Build HTML snippet html1 | Code in Python (Beta) | Merge |  |
| Loop Over Items4 | splitInBatches | Batch loop controller (Movies path) | Switch; RSS Read2 | Code in Python (Beta)1; RSS Read2 | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| RSS Read2 | rssFeedRead | Fetch RSS items (Movies) | Loop Over Items4 | Loop Over Items4 | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Code in Python (Beta)1 | code (python) | Keep first 3 items (Movies) | Loop Over Items4 | Movie | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Movie | code (python) | Build HTML snippet html2 | Code in Python (Beta)1 | Merge |  |
| Loop Over Items5 | splitInBatches | Batch loop controller (Games path) | Switch; RSS Read3 | Code in Python (Beta)2; RSS Read3 | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| RSS Read3 | rssFeedRead | Fetch RSS items (Games) | Loop Over Items5 | Loop Over Items5 | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Code in Python (Beta)2 | code (python) | Keep first 3 items (Games) | Loop Over Items5 | Jeu video | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Jeu video | code (python) | Build HTML snippet html3 | Code in Python (Beta)2 | Merge |  |
| Merge | merge | Join the 3 category snippets | Manga; Movie; Jeu video | Aggregate | ## 📦 Combine & Convert\nMerge all feeds and convert to CSV for AI |
| Aggregate | aggregate | Aggregate snippets into $json.HTML array | Merge | HTML Template | ## 📦 Combine & Convert\nMerge all feeds and convert to CSV for AI |
| HTML Template | html | Render full newsletter HTML | Aggregate | Mail Campaign; Send a Mail | ## Set HTML part \n#Add information in HTML code part |
| Mail Campaign | sendInBlue | Send HTML email via Brevo | HTML Template | — | ## 📧 Set HTML FileNewsletter Delivery\nFormat and send the personalized newsletter. |
| Send a Mail | gmail | Send HTML email via Gmail | HTML Template | — | ## 📧 Set HTML FileNewsletter Delivery\nFormat and send the personalized newsletter. |
| Sticky Note1 | stickyNote | Canvas documentation | — | — | ## Daily trigger |
| Sticky Note - Processing1 | stickyNote | Canvas documentation | — | — | ## 🔄 RSS Feed Processing\nSplit, loop through feeds in batches, and fetch content |
| Sticky Note - Sources1 | stickyNote | Canvas documentation | — | — | ## 📰 RSS Feed Sources\nDefine which RSS feeds to monitor for tech news and tools |
| Sticky Note - Combine1 | stickyNote | Canvas documentation | — | — | ## Set HTML part \n#Add information in HTML code part |
| Sticky Note - Combine2 | stickyNote | Canvas documentation | — | — | ## 📦 Combine & Convert\nMerge all feeds and convert to CSV for AI |
| Sticky Note - Email1 | stickyNote | Canvas documentation | — | — | ## 📧 Set HTML FileNewsletter Delivery\nFormat and send the personalized newsletter. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: `RSS par Catégorie vers HTML`.

2. **Add a Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure interval:
     - Every **5 hours**
     - At **minute 30**
   - This is the workflow entry point.

3. **Add a Set node to define categories**
   - Node: **Set** (rename to `Category Rss`)
   - Mode: **Raw**
   - JSON output with a `cat` array (include at least):
     - `"Manga"`, `"Cinéma"`, `"Jeux Vidéo"`
     - (Optional: other categories, but they won’t route unless you add Switch rules)

4. **Split the category array**
   - Node: **Split Out** (rename to `Split Out2`)
   - Field to split out: `cat`
   - Connect: `Schedule Trigger` → `Category Rss` → `Split Out2`

5. **Create/prepare Google Sheets “RSS database”**
   - Spreadsheet must contain columns:
     - `Catégorie`
     - `Lien RSS`
   - Each row: a category value and an RSS URL.

6. **Add Google Sheets node to query feed sources**
   - Node: **Google Sheets** (rename to `Rss database`)
   - Credential: **Google Sheets OAuth2**
   - Document: select your spreadsheet
   - Sheet: select the target sheet (e.g., Sheet1)
   - Add a filter:
     - Column: `Catégorie`
     - Equals: `{{$json.cat}}`
   - Connect: `Split Out2` → `Rss database`

7. **Add a Switch to route by category**
   - Node: **Switch** (rename `Switch`)
   - Create 3 rules (string equals):
     - `{{$json["Catégorie"]}}` equals `Manga`
     - equals `Cinéma`
     - equals `Jeux Vidéo`
   - Connect: `Rss database` → `Switch`

8. **Manga pipeline (RSS → limit → HTML snippet)**
   1) Add **Split In Batches** (rename `Loop Over Items3`)
      - Batch size: `2`
   2) Add **RSS Feed Read** (rename `RSS Read1`)
      - URL: `{{$json["Lien RSS"]}}`
      - Set: “Continue on Fail” / `onError: continue` (in UI)
   3) Add **Code** node in Python (rename `Code in Python (Beta)`)
      - Return first 3 items:
        - `return _input.all()[:3]`
   4) Add **Code** node in Python (rename `Manga`)
      - Build `html1` snippet (article cards) and return:
        - `return [{"HTML": {"html1": m}}]`
   5) Wire it as in the workflow:
      - `Switch` (Manga output) → `Loop Over Items3`
      - `Loop Over Items3` output 1 → `RSS Read1`
      - `RSS Read1` → `Loop Over Items3` (to continue)
      - `Loop Over Items3` output 0 → `Code in Python (Beta)` → `Manga`

9. **Movies pipeline**
   - Repeat the same structure:
     - `Loop Over Items4` (SplitInBatches, batchSize 2)
     - `RSS Read2` (URL `{{$json["Lien RSS"]}}`, continue on fail)
     - `Code in Python (Beta)1` (`return _input.all()[:3]`)
     - `Movie` code node returning `{"HTML":{"html2": m}}`
   - Connect Switch “Cinéma” output to `Loop Over Items4`.

10. **Games pipeline**
   - Repeat:
     - `Loop Over Items5`, `RSS Read3`, `Code in Python (Beta)2`, `Jeu video` returning `{"HTML":{"html3": m}}`
   - Connect Switch “Jeux Vidéo” output to `Loop Over Items5`.

11. **Merge the 3 HTML snippets**
   - Add **Merge** node (rename `Merge`)
   - Set number of inputs: `3`
   - Connect:
     - `Manga` → `Merge` input 1
     - `Movie` → `Merge` input 2
     - `Jeu video` → `Merge` input 3

12. **Aggregate into a single item**
   - Add **Aggregate** node (rename `Aggregate`)
   - Mode: aggregate all item data
   - Destination field name: `HTML`
   - Connect: `Merge` → `Aggregate`

13. **Render full HTML**
   - Add **HTML** node (rename `HTML Template`)
   - Paste your HTML template
   - Insert expressions to inject snippets (matching your aggregate order), for example:
     - `{{ $json.HTML[0].HTML.html1 }}` etc.
   - Connect: `Aggregate` → `HTML Template`

14. **Send via Brevo (Sendinblue)**
   - Add node: **Sendinblue / Brevo** (rename `Mail Campaign`)
   - Credential: **SendInBlue API**
   - Enable HTML content
   - Set:
     - Sender email
     - Recipient(s)
     - Subject
     - HTML content: `{{$json.html}}` (ensure the HTML node outputs `html`)
   - Connect: `HTML Template` → `Mail Campaign`

15. **Send via Gmail**
   - Add node: **Gmail** (rename `Send a Mail`)
   - Credential: **Gmail OAuth2**
   - To, Subject, Message:
     - Message: `{{$json.html}}`
   - Connect: `HTML Template` → `Send a Mail`

16. **Validate outputs**
   - Run once manually.
   - Confirm:
     - Google Sheets returns correct rows.
     - RSS nodes return items with `title/link/content`.
     - Merge/Aggregate order matches template indexes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The HTML template includes JavaScript tab switching (`<script>`). Most email clients block scripts, so tabs will likely not work in real inboxes. | Email rendering limitation (Gmail, Outlook, etc.) |
| Category list contains many categories, but Switch only routes Manga/Cinéma/Jeux Vidéo. Other categories are ignored unless additional Switch rules and pipelines are added. | Logic consistency |
| Google Sheets must include columns `Catégorie` and `Lien RSS` with exact spelling/accents. | Integration requirement |
| Spreadsheet referenced in node cache: “Rss Database 2” | https://docs.google.com/spreadsheets/d/1anl-6-nbOfrsXJc7fhAAZ0yIONrqkg7qB7MXrqK37pw/edit?usp=drivesdk |