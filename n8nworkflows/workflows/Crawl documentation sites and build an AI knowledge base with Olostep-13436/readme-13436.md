Crawl documentation sites and build an AI knowledge base with Olostep

https://n8nworkflows.xyz/workflows/crawl-documentation-sites-and-build-an-ai-knowledge-base-with-olostep-13436


# Crawl documentation sites and build an AI knowledge base with Olostep

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Documentation Crawler  
**Purpose:** Crawl a documentation site, scrape each page into markdown, extract structured API metadata with an LLM, rewrite the page into a polished “developer reference article”, and store each rewritten page as a Google Doc inside a Google Drive folder hierarchy mirroring the site’s URL structure.

**Primary use cases**
- Building/maintaining an internal knowledge base from public docs
- Creating structured AI-ingestible documentation libraries
- Periodic refresh of vendor API docs into a controlled format (Google Docs)

### 1.1 Logical Blocks (by dependency)
1. **Entry & Sitemap Discovery**
   - Manual trigger → Olostep “map” crawl → list of URLs
2. **Service Identification & Parent Folder Setup**
   - Derive “Service” name from URL → create a parent Drive folder
3. **URL List Preparation & Merge**
   - Split sitemap URLs into items → merge with parent folder metadata
4. **Batch Loop & Folder Hierarchy Builder**
   - Iterate URLs in batches → compute path “counter” → check/create subfolders until leaf depth
5. **Scrape Page & AI Processing**
   - Scrape markdown → extract metadata (Gemini) → merge metadata + content → wait (throttle) → writer agent (Gemini)
6. **Document Output**
   - Create Google Doc in correct folder → insert generated article → continue loop

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Sitemap Discovery
**Overview:** Starts manually and uses Olostep to crawl the root docs site, producing a sitemap-like list of URLs to process.

**Nodes involved:**  
- When clicking ‘Execute workflow’  
- Create a map

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (entry point)
- **Config:** No parameters; user runs it on demand.
- **Outputs:** Triggers the workflow once.
- **Connections:** → `Create a map`
- **Failure/edge cases:** None (except n8n execution constraints).

**Sticky note content (applies in this area):**  
(From “AI Documentation Crawler & Knowledge Base Builder” large note — general workflow description; see Section 5.)

#### Node: Create a map
- **Type / role:** `n8n-nodes-olostep.olostepScrape` (Olostep Scrape) — **resource: map**
- **Config choices:**
  - **URL:** `https://docs.tavily.com` (root documentation URL to crawl)
  - `top_n` left empty (so defaults apply; may return many URLs)
- **Credentials:** Olostep Scrape account
- **Outputs:** A JSON payload containing discovered documentation URLs (referenced later as `urls`).
- **Connections:** → `Service Name` and → `Split Out1`
- **Failure/edge cases:**
  - Olostep auth errors / invalid API key
  - Crawl limits, rate limits, or timeouts on large doc sites
  - Site blocks crawlers (robots/WAF), resulting in partial URL list
  - Output schema changes: downstream expects `urls` field

**Sticky note (Creates a Map):**  
“Crawls the root documentation URL and generates a full sitemap. Returns all discoverable documentation URLs under the given domain.”

---

### Block 2 — Service Identification & Parent Folder Setup
**Overview:** Derives a “Service” label from a URL and creates a top-level Google Drive folder under a fixed base folder.

**Nodes involved:**  
- Service Name  
- Create Parent Folder  
- Edit Fields

#### Node: Service Name
- **Type / role:** Set node — derive service identifier
- **Key expression:**
  - `Service = {{ $json.url.split('.')[1] }}`
- **Interpretation:** Assumes the incoming item has a `url` field and that splitting by `.` yields something like `docs.tavily.com` → index `[1]` = `tavily`.
- **Connections:** → `Create Parent Folder`
- **Failure/edge cases:**
  - If input URL is missing or not shaped as expected (e.g., `developer.company.co.uk`), `[1]` may be wrong.
  - If the “map” output doesn’t contain `url` at the top level (only `urls` array), this expression can resolve to `undefined`.

**Sticky note (Creates Parent Folder area):**  
“Creates a parent folder for this service in Google Drive. All generated API reference documents will be stored here.”

#### Node: Create Parent Folder
- **Type / role:** Google Drive — create folder
- **Config choices:**
  - **Resource:** folder
  - **Folder name:** `{{ $json.Service }}`
  - **Parent folder ID:** fixed folder `api workflow`  
    - Cached URL: https://drive.google.com/drive/folders/1ogVqaOeuMSGOs_jUclfs8Mj6oKX71EhZ
  - **Drive:** “My Drive”
- **Credentials:** Google Drive OAuth2
- **Outputs:** Folder metadata including `id` (used later as parent folder id).
- **Connections:** → `Edit Fields`
- **Failure/edge cases:**
  - OAuth permission issues (cannot create folder in target parent)
  - Folder name collisions (Drive allows duplicates; later logic assumes IDs, so duplicates can fragment output)
  - If `Service` empty, folder may be created with blank/odd name

#### Node: Edit Fields
- **Type / role:** Set node — normalize required fields for downstream merge
- **Sets:**
  - `parent_id = {{ $json.id }}` (Drive folder id returned from Create Parent Folder)
  - `counter = 3` (initial URL path depth index)
- **Connections:** → `Merge` (input 1)
- **Failure/edge cases:**
  - If Drive node didn’t return `id`, `parent_id` will be empty and all folder/file operations will fail.

---

### Block 3 — URL List Preparation & Merge
**Overview:** Splits the sitemap URL list into individual items and merges each URL item with the service folder metadata (`parent_id`, `counter`).

**Nodes involved:**  
- Split Out1  
- Merge

#### Node: Split Out1
- **Type / role:** Split Out — creates one item per URL
- **Config:**
  - **Field to split out:** `urls`
- **Expected input:** From `Create a map` output containing an array field `urls`.
- **Outputs:** Items where each contains the split value (the node later references `$('Loop Over Items').item.json.urls`, implying the URL is stored under key `urls`).
- **Connections:** → `Merge` (input 0)
- **Failure/edge cases:**
  - If `urls` is not an array, node errors or produces no items.
  - If output item structure differs (e.g., the URL is put under `url`), downstream expressions break.

**Sticky note (Merge):**  
“Merges service metadata with the list of documentation URLs so both are available during processing.”

#### Node: Merge
- **Type / role:** Merge node — combine streams
- **Mode:** `combine` with `combineAll`
- **Inputs:**
  - Input 0: URL items from `Split Out1`
  - Input 1: Metadata (`parent_id`, `counter`) from `Edit Fields`
- **Outputs:** Each URL item augmented with `parent_id` and `counter`.
- **Connections:** → `Loop Over Items`
- **Failure/edge cases:**
  - If one input stream is empty, result may be empty (depending on merge behavior/version).
  - Ensure the merge creates the intended Cartesian or “attach metadata to all” effect; `combineAll` can produce multiple combinations if both inputs have multiple items.

---

### Block 4 — Batch Loop & Folder Hierarchy Builder
**Overview:** Iterates each URL, creates Drive subfolders for each path segment (if missing), and stops when reaching a “leaf” where scraping should occur.

**Nodes involved:**  
- Loop Over Items  
- No Operation, do nothing  
- counter  
- If1  
- Search files and folders  
- If  
- Create folder  
- counter++  
- counter++1

#### Node: Loop Over Items
- **Type / role:** Split In Batches — loop controller
- **Config:** Default options (batch size default; in n8n it’s typically 1 unless set).
- **Connections:**
  - Output 0 → `No Operation, do nothing`
  - Output 1 → `counter` (this is the “continue” output for next batch iteration in many patterns; here both are used)
- **Failure/edge cases:**
  - Misconfigured batch size can cause excessive API calls or memory usage.
  - If the loop is not correctly “fed back” it can stop prematurely; here the loop continues via `Update a document` → `Loop Over Items`.

#### Node: No Operation, do nothing
- **Type / role:** NoOp — placeholder
- **Config:** none
- **Connections:** none downstream (in this workflow it does not contribute to logic)
- **Failure/edge cases:** none
- **Note:** This node is effectively unused for processing.

**Sticky note (Counter):**  
“Tracks the current URL depth being processed. Used to dynamically create nested folder structures that match the doc hierarchy. After that checks whether additional URL path segments exist. Determines whether to create a subfolder or proceed with scraping.”

#### Node: counter
- **Type / role:** Set node — prepares fields for the folder logic
- **Sets:**
  - `counter = {{ $json.counter }}`
  - `url = {{ $('Loop Over Items').item.json.urls }}`
  - `parent_id = {{ $json.parent_id }}`
- **Interpretation:** Pulls the URL from the batch item’s `urls` field and carries forward the current `counter` and `parent_id`.
- **Connections:** → `If1`
- **Failure/edge cases:**
  - If batch item uses a different property name than `urls`, `url` becomes empty.
  - Counter type inconsistency: later nodes sometimes set `counter` as **string**, sometimes **number**.

#### Node: If1
- **Type / role:** IF — decide whether to keep building folders or scrape now
- **Conditions (exists checks):**
  - `{{ $json.url.split('/')[$json.counter] }}` exists
  - `{{ $json.url.split('/')[$json.counter + 1] }}` exists  
- **Interpretation:** If there is another path segment after the current one, continue folder creation; if not, treat as leaf and scrape.
- **Outputs:**
  - **True** → `Search files and folders`
  - **False** → `set data` (scrape path)
- **Failure/edge cases:**
  - The “exists” operator is used with a `rightValue` set (e.g., `4` / `""`) in JSON; in n8n IF v2+ this may be ignored but can be confusing.
  - URLs with trailing slashes can create empty last segments (`""`) which changes the leaf decision.
  - Query strings/fragments (`?x=y`, `#section`) may appear in the last segment; folder names may become messy.

#### Node: Search files and folders
- **Type / role:** Google Drive — search for a folder under current parent
- **Config choices:**
  - **Resource:** fileFolder
  - **Limit:** 1
  - **Filter:** `folderId = {{ $json.parent_id }}` (search within current parent folder)
  - **Query string:** `{{ $json.url.split('/')[$json.counter] }}` (current path segment name)
  - `alwaysOutputData: true` and `retryOnFail: true` (maxTries 2)
- **Connections:** → `If`
- **Failure/edge cases:**
  - Drive search can match files as well as folders (resource is fileFolder); ensure results are folders when expected.
  - Special characters in path segment may not match expected Drive search behavior.
  - If no result, output still exists due to `alwaysOutputData`; `id` will be missing.

**Sticky note (Search files/folders):**  
“Checks if a folder for the current documentation section already exists. Prevents duplicate folder creation.”

#### Node: If
- **Type / role:** IF — decide whether folder exists
- **Condition:** `{{ $json.id }}` exists
- **Outputs:**
  - **True** (folder found) → `counter++1`
  - **False** (not found) → `Create folder`
- **Failure/edge cases:**
  - If Drive search returns a file’s id (not folder), you may incorrectly treat it as folder; subsequent folder creation under that id will fail.

#### Node: Create folder
- **Type / role:** Google Drive — create subfolder for this path segment
- **Config choices:**
  - **Name:** `{{ $('counter').item.json.url.split('/')[ $('counter').item.json.counter ] }}`
  - **Parent folder:** `{{ $('counter').item.json.parent_id }}`
  - **Drive:** My Drive
  - `retryOnFail: true` (maxTries 2)
- **Connections:** → `counter++`
- **Failure/edge cases:**
  - Name can be empty if URL segment is empty.
  - Invalid characters may be normalized by Drive, causing mismatches later.

**Sticky note (Create Folder):**  
“Creates a subfolder that mirrors the documentation URL structure. Keeps generated docs neatly organized by section. Increments the URL depth counter. Moves the workflow deeper into the documentation structure.”

#### Node: counter++
- **Type / role:** Set node — advance depth after folder creation
- **Sets:**
  - `counter = {{ $('counter').item.json.counter + 1 }}`
  - `parent_id = {{ $json.id }}` (newly created folder id)
- **Connections:** → `counter` (loop back into If1 decision)
- **Failure/edge cases:**
  - `counter` is configured as **string** type in this node, but used numerically later. This can cause string concatenation bugs in other contexts; in expressions here it often still works, but it’s risky.

#### Node: counter++1
- **Type / role:** Set node — advance depth when folder already exists
- **Sets:**
  - `counter = {{ $('counter').item.json.counter + 1 }}`
  - `parent_id = {{ $json.id }}` (existing folder id found by search)
- **Connections:** → `counter`
- **Failure/edge cases:** same counter type concern as above.

---

### Block 5 — Scrape Page & AI Processing
**Overview:** Once at a leaf URL, scrape markdown content, extract structured metadata with Gemini, merge metadata with content, throttle, and generate a rewritten developer reference article.

**Nodes involved:**  
- set data  
- Scrape a URL  
- Information Extractor  
- Google Gemini Chat Model2  
- Merge1  
- Wait  
- AI Agent  
- Google Gemini Chat Model

#### Node: set data
- **Type / role:** Set node — normalize fields before scraping
- **Sets:**
  - `counter = {{ $json.counter }}`
  - `url = {{ $json.url }}`
  - `parent_id = {{ $json.parent_id }}`
- **Connections:** → `Scrape a URL`
- **Failure/edge cases:** If upstream uses `urls` not `url` at that point, URL may be missing. (In this workflow, `counter` node sets `url`, and If1 false branch goes to `set data`, so it should exist.)

**Sticky note (Scrape URL):**  
“Scrapes the full documentation page content. Extracts clean markdown content for AI processing.”

#### Node: Scrape a URL
- **Type / role:** Olostep Scrape — scrape a single URL
- **Config:**
  - `url_to_scrape = {{ $json.url }}`
  - `retryOnFail: true`, `maxTries: 2`
- **Outputs:** Includes `markdown_content` (used by LLM nodes) and likely other scrape metadata.
- **Connections:**
  - → `Information Extractor`
  - → `Merge1` (input 1)
- **Failure/edge cases:**
  - Network/timeouts, 403/WAF blocks, incomplete markdown extraction
  - Very large pages may exceed LLM/context constraints downstream

#### Node: Information Extractor
- **Type / role:** LangChain Information Extractor — structured extraction from text
- **Model:** Provided via `Google Gemini Chat Model2` connection
- **Input text:** `{{ $json.markdown_content }}`
- **Extracted attributes (required):**
  - Curl
  - Quick Summary
  - Authentication
  - Common Gotchas
- **Connections:** → `Merge1` (input 0)
- **Failure/edge cases:**
  - If `markdown_content` missing/empty, extraction fails or returns empty outputs
  - Model may hallucinate if the page is not an endpoint doc
  - Token limits: large markdown may be truncated by the provider
  - Auth/rate limits from Gemini

**Sticky note (Information extractor):**  
“Uses AI to extract structured metadata from the documentation: Curl example, endpoint summary, authentication method, and common pitfalls.”

#### Node: Google Gemini Chat Model2
- **Type / role:** Gemini chat model connector for the extractor
- **Credentials:** Gemini(PaLM) API
- **Connections:** Supplies the `ai_languageModel` input to `Information Extractor`
- **Failure/edge cases:** Credential/quotas; region availability.

#### Node: Merge1
- **Type / role:** Merge — combine extraction output with raw scrape context
- **Mode:** `combineByPosition`
- **Inputs:**
  - Input 0: `Information Extractor` output (structured fields under `$json.output...`)
  - Input 1: `Scrape a URL` output (raw markdown, url, etc.)
- **Connections:** → `Wait`
- **Failure/edge cases:**
  - If one branch errors or returns zero items, “by position” merge can misalign or produce empty output.

#### Node: Wait
- **Type / role:** Wait/throttle
- **Config:** wait **1 minute**
- **Connections:** → `AI Agent`
- **Failure/edge cases:** Slows processing; for large crawls it can significantly extend runtime (but reduces throttling risk).

#### Node: AI Agent
- **Type / role:** LangChain Agent — produces final rewritten article
- **Model:** Provided via `Google Gemini Chat Model`
- **Prompt input (`text`):** A composed markdown including:
  - Extracted metadata from `Information Extractor` (`$json.output.Curl`, etc.)
  - URL (`$json.url`)
  - Raw documentation (`$json.markdown_content`)
- **System message:** Enforces article structure (title, TL;DR quick start, implementation guide, parameter table, deep dive, gotchas, response examples) and style (scannable, action-oriented).
- **Connections:** → `Create a document`
- **Failure/edge cases:**
  - Output may be too long for Google Docs insertion limits in one operation
  - Model can produce invalid markdown or inconsistent formatting
  - If extractor output fields are missing, prompt template renders blanks

**Sticky note (Writer Agent):**  
“Converts raw documentation + extracted metadata into a clean, developer-friendly API reference article.”

#### Node: Google Gemini Chat Model
- **Type / role:** Gemini chat model connector for the writer agent
- **Credentials:** Gemini(PaLM) API
- **Connections:** Supplies `ai_languageModel` input to `AI Agent`
- **Failure/edge cases:** Same as Gemini2; also cost can be high for many pages.

---

### Block 6 — Document Output (Google Docs)
**Overview:** Creates a Google Doc per page in the computed Drive folder and inserts the generated reference article content, then continues to next URL.

**Nodes involved:**  
- Create a document  
- Update a document

#### Node: Create a document
- **Type / role:** Google Docs — create document
- **Config choices:**
  - **Title:** `{{ $('set data').item.json.url.split('/')[ $('set data').item.json.counter ] }}`
    - Uses the current URL segment as document title.
  - **Folder ID:** `{{ $('set data').item.json.parent_id }}`
- **Credentials:** Google Docs OAuth2
- **Connections:** → `Update a document`
- **Failure/edge cases:**
  - Title may be empty or include query strings
  - Permission issues writing into that folder
  - Google Docs API quota

**Sticky note (Create a Document):**  
“Creates a new Google Doc for the current documentation page and update that document with the full context. The document title matches the URL path segment.”

#### Node: Update a document
- **Type / role:** Google Docs — update/insert content
- **Operation:** update (non-simple mode)
- **Action:** insert text = `{{ $('AI Agent').item.json.output }}`
- **Document URL/ID:** `{{ $json.id }}` (id returned by Create a document)
- **Connections:** → `Loop Over Items` (to process the next URL in the batch loop)
- **Failure/edge cases:**
  - Very large insert text may fail (Docs API limits)
  - Formatting: inserting markdown as plain text won’t render as rich formatting unless further conversion is used
  - If `AI Agent` output contains unsupported characters or extremely long code blocks, insertion can error

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry point | — | Create a map | # AI Documentation Crawler & Knowledge Base Builder … (general description; see Notes) |
| Create a map | olostepScrape (map) | Crawl root docs URL → sitemap URLs | When clicking ‘Execute workflow’ | Service Name; Split Out1 | ## Creates a Map / Crawls the root documentation URL and generates a full sitemap. Returns all discoverable documentation URLs under the given domain. |
| Service Name | set | Derive service name from URL | Create a map | Create Parent Folder | ## Creates Parent Folder / Creates a parent folder for this service in Google Drive. All generated API reference documents will be stored here. |
| Create Parent Folder | googleDrive (folder) | Create service parent folder in Drive | Service Name | Edit Fields | ## Creates Parent Folder / Creates a parent folder for this service in Google Drive. All generated API reference documents will be stored here. |
| Edit Fields | set | Store parent folder id + init counter | Create Parent Folder | Merge | ## Creates Parent Folder / Creates a parent folder for this service in Google Drive. All generated API reference documents will be stored here. |
| Split Out1 | splitOut | Split `urls[]` into individual items | Create a map | Merge | ## Merge / Merges service metadata with the list of documentation URLs so both are available during processing. |
| Merge | merge | Combine URL items with service metadata | Split Out1; Edit Fields | Loop Over Items | ## Merge / Merges service metadata with the list of documentation URLs so both are available during processing. |
| Loop Over Items | splitInBatches | Iterate each URL item | Merge; Update a document | No Operation, do nothing; counter | ## Counter / Tracks the current URL depth being processed… |
| No Operation, do nothing | noOp | Placeholder branch (no logic) | Loop Over Items | — | ## Counter / Tracks the current URL depth being processed… |
| counter | set | Prepare `url`, `parent_id`, `counter` for path processing | Loop Over Items; counter++; counter++1 | If1 | ## Counter / Tracks the current URL depth being processed… |
| If1 | if | Decide: create more folders vs scrape | counter | Search files and folders; set data | ## Counter / Tracks the current URL depth being processed… |
| Search files and folders | googleDrive (fileFolder) | Search if subfolder exists | If1 | If | ## Search files/folders / Checks if a folder for the current documentation section already exists. Prevents duplicate folder creation. |
| If | if | Branch on folder existence | Search files and folders | counter++1; Create folder | ## Search files/folders / Checks if a folder for the current documentation section already exists. Prevents duplicate folder creation. |
| Create folder | googleDrive (folder) | Create missing subfolder | If | counter++ | ## Create Folder / Creates a subfolder that mirrors the documentation URL structure… |
| counter++ | set | Increment depth; move parent to new folder | Create folder | counter | ## Create Folder / Creates a subfolder that mirrors the documentation URL structure… |
| counter++1 | set | Increment depth; move parent to existing folder | If | counter | ## Create Folder / Creates a subfolder that mirrors the documentation URL structure… |
| set data | set | Normalize leaf URL data for scraping | If1 | Scrape a URL | ## Scrape URL / Scrapes the full documentation page content. Extracts clean markdown content for AI processing. |
| Scrape a URL | olostepScrape | Scrape page → markdown | set data | Information Extractor; Merge1 | ## Scrape URL / Scrapes the full documentation page content. Extracts clean markdown content for AI processing. |
| Google Gemini Chat Model2 | lmChatGoogleGemini | LLM backend for extraction | — | Information Extractor (ai_languageModel) | ## Information extractor / Uses AI to extract structured metadata… |
| Information Extractor | informationExtractor | Extract Curl/Summary/Auth/Gotchas | Scrape a URL | Merge1 | ## Information extractor / Uses AI to extract structured metadata… |
| Merge1 | merge | Combine scrape + extracted metadata | Information Extractor; Scrape a URL | Wait | ## Writer Agent / Converts raw documentation + extracted metadata into a clean, developer-friendly API reference article. |
| Wait | wait | Throttle to reduce rate limiting | Merge1 | AI Agent | ## Writer Agent / Converts raw documentation + extracted metadata into a clean, developer-friendly API reference article. |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM backend for writer agent | — | AI Agent (ai_languageModel) | ## Writer Agent / Converts raw documentation + extracted metadata into a clean, developer-friendly API reference article. |
| AI Agent | agent | Generate final “developer reference article” | Wait | Create a document | ## Writer Agent / Converts raw documentation + extracted metadata into a clean, developer-friendly API reference article. |
| Create a document | googleDocs | Create doc in target folder | AI Agent | Update a document | ## Create a Document / Creates a new Google Doc… |
| Update a document | googleDocs | Insert generated content into doc | Create a document | Loop Over Items | ## Create a Document / Creates a new Google Doc… |
| Sticky Note | stickyNote | Comment block | — | — | (N/A — this is itself the note content) |
| Sticky Note1 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note2 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note3 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note4 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note5 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note6 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note7 | stickyNote | Empty comment block | — | — | (blank) |
| Sticky Note8 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note9 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note10 | stickyNote | Comment block | — | — | (N/A) |
| Sticky Note11 | stickyNote | Comment block | — | — | (N/A) |

> Note: Sticky Note nodes are included as nodes (per requirement “do not skip any nodes”), but their content is already reflected in relevant rows above. Sticky Note7 is empty.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Documentation Crawler**

2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add Olostep “map” crawler**
   - Node: **Olostep Scrape**
   - Name: `Create a map`
   - Resource: **map**
   - URL: `https://docs.tavily.com` (replace with your docs root)
   - Credentials: create **Olostep Scrape account** credential (API key)

4. **Split the discovered URLs**
   - Node: **Split Out**
   - Name: `Split Out1`
   - Field to split out: `urls`
   - Connect: `Create a map` → `Split Out1`

5. **Derive service name**
   - Node: **Set**
   - Name: `Service Name`
   - Add field:
     - `Service` (string) = `{{ $json.url.split('.')[1] }}`
   - Connect: `Create a map` → `Service Name`
   - (If your Olostep map output does not expose `url`, change this to use your known root domain or the first element of `urls`.)

6. **Create the parent Drive folder**
   - Node: **Google Drive**
   - Name: `Create Parent Folder`
   - Resource: **folder**
   - Name: `{{ $json.Service }}`
   - Parent folder: choose the base folder where you want everything stored (in the template: “api workflow”)
   - Credentials: **Google Drive OAuth2** with permission to create folders there
   - Connect: `Service Name` → `Create Parent Folder`

7. **Prepare metadata (`parent_id`, `counter`)**
   - Node: **Set**
   - Name: `Edit Fields`
   - Fields:
     - `parent_id` (string) = `{{ $json.id }}`
     - `counter` (number) = `3`
   - Connect: `Create Parent Folder` → `Edit Fields`

8. **Merge URL items with metadata**
   - Node: **Merge**
   - Name: `Merge`
   - Mode: **Combine**
   - Combine by: **combineAll**
   - Connect:
     - `Split Out1` → `Merge` (Input 0)
     - `Edit Fields` → `Merge` (Input 1)

9. **Add loop controller**
   - Node: **Split In Batches**
   - Name: `Loop Over Items`
   - (Optional) Set batch size explicitly (commonly 1)
   - Connect: `Merge` → `Loop Over Items`

10. **(Optional) Add NoOp placeholder**
    - Node: **No Operation**
    - Name: `No Operation, do nothing`
    - Connect: `Loop Over Items` (Output 0) → `No Operation, do nothing`

11. **Add “counter” set node**
    - Node: **Set**
    - Name: `counter`
    - Fields:
      - `counter` (number) = `{{ $json.counter }}`
      - `url` (string) = `{{ $('Loop Over Items').item.json.urls }}`
      - `parent_id` (string) = `{{ $json.parent_id }}`
    - Connect: `Loop Over Items` (Output 1) → `counter`

12. **Add IF to decide folder-creation vs scrape (leaf detection)**
    - Node: **IF**
    - Name: `If1`
    - Conditions (String → exists):
      - Left: `{{ $json.url.split('/')[$json.counter] }}`
      - Left: `{{ $json.url.split('/')[$json.counter + 1] }}`
    - Connect: `counter` → `If1`

13. **Search for existing subfolder**
    - Node: **Google Drive**
    - Name: `Search files and folders`
    - Resource: **fileFolder**
    - Filter: Folder ID = `{{ $json.parent_id }}`
    - Query String = `{{ $json.url.split('/')[$json.counter] }}`
    - Limit: 1
    - Enable: **Retry on fail**, **Always output data**
    - Connect: `If1` (true) → `Search files and folders`

14. **IF folder exists**
    - Node: **IF**
    - Name: `If`
    - Condition: `{{ $json.id }}` exists
    - Connect: `Search files and folders` → `If`

15. **Create folder if missing**
    - Node: **Google Drive**
    - Name: `Create folder`
    - Resource: **folder**
    - Parent folder ID: `{{ $('counter').item.json.parent_id }}`
    - Name: `{{ $('counter').item.json.url.split('/')[ $('counter').item.json.counter ] }}`
    - Enable retries
    - Connect: `If` (false) → `Create folder`

16. **Increment counter after creating folder**
    - Node: **Set**
    - Name: `counter++`
    - Fields:
      - `counter` = `{{ $('counter').item.json.counter + 1 }}`
      - `parent_id` = `{{ $json.id }}`
    - Connect: `Create folder` → `counter++`
    - Connect: `counter++` → `counter`

17. **Increment counter if folder already exists**
    - Node: **Set**
    - Name: `counter++1`
    - Fields:
      - `counter` = `{{ $('counter').item.json.counter + 1 }}`
      - `parent_id` = `{{ $json.id }}`
    - Connect: `If` (true) → `counter++1`
    - Connect: `counter++1` → `counter`

18. **Prepare scraping input when at leaf**
    - Node: **Set**
    - Name: `set data`
    - Fields:
      - `counter` (number) = `{{ $json.counter }}`
      - `url` (string) = `{{ $json.url }}`
      - `parent_id` (string) = `{{ $json.parent_id }}`
    - Connect: `If1` (false) → `set data`

19. **Scrape the leaf URL**
    - Node: **Olostep Scrape**
    - Name: `Scrape a URL`
    - URL to scrape: `{{ $json.url }}`
    - Enable retries
    - Connect: `set data` → `Scrape a URL`

20. **Add Gemini model for extraction**
    - Node: **Google Gemini Chat Model**
    - Name: `Google Gemini Chat Model2`
    - Credentials: **Gemini(PaLM) API**

21. **Add Information Extractor**
    - Node: **Information Extractor**
    - Name: `Information Extractor`
    - Text: `{{ $json.markdown_content }}`
    - Define attributes (all required):
      - Curl
      - Quick Summary
      - Authentication
      - Common Gotchas
    - Connect: `Scrape a URL` → `Information Extractor`
    - Connect AI model: `Google Gemini Chat Model2` (ai_languageModel) → `Information Extractor`

22. **Merge extracted metadata with scrape output**
    - Node: **Merge**
    - Name: `Merge1`
    - Mode: **combineByPosition**
    - Connect:
      - `Information Extractor` → `Merge1` (Input 0)
      - `Scrape a URL` → `Merge1` (Input 1)

23. **Throttle**
    - Node: **Wait**
    - Name: `Wait`
    - Unit: minutes, Amount: 1
    - Connect: `Merge1` → `Wait`

24. **Add Gemini model for writing**
    - Node: **Google Gemini Chat Model**
    - Name: `Google Gemini Chat Model`
    - Credentials: **Gemini(PaLM) API**

25. **Add Writer Agent**
    - Node: **AI Agent**
    - Name: `AI Agent`
    - Prompt type: define
    - System message: use the same role/instructions as in the workflow (expert technical writer; required sections).
    - Text template should include:
      - Extracted metadata: `{{ $json.output.Curl }}`, etc.
      - URL: `{{ $json.url }}`
      - Raw docs: `{{ $json.markdown_content }}`
    - Connect: `Wait` → `AI Agent`
    - Connect AI model: `Google Gemini Chat Model` (ai_languageModel) → `AI Agent`

26. **Create Google Doc**
    - Node: **Google Docs**
    - Name: `Create a document`
    - Title: `{{ $('set data').item.json.url.split('/')[ $('set data').item.json.counter ] }}`
    - Folder ID: `{{ $('set data').item.json.parent_id }}`
    - Credentials: **Google Docs OAuth2**
    - Connect: `AI Agent` → `Create a document`

27. **Insert content into the doc**
    - Node: **Google Docs**
    - Name: `Update a document`
    - Operation: **Update**
    - Use action “insert text” with: `{{ $('AI Agent').item.json.output }}`
    - Document URL/ID: `{{ $json.id }}`
    - Connect: `Create a document` → `Update a document`

28. **Continue the loop**
    - Connect: `Update a document` → `Loop Over Items`  
      (This is what advances to the next URL batch item.)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Documentation Crawler & Knowledge Base Builder… crawls technical documentation websites, scrapes content, converts it into clean structured developer-friendly documentation… saved as Google Docs… includes rate control… customization ideas (Notion/Confluence/vector DB), scheduling re-crawls.” | Sticky note (general workflow description) |
| Base storage folder (“api workflow”) in Google Drive | https://drive.google.com/drive/folders/1ogVqaOeuMSGOs_jUclfs8Mj6oKX71EhZ |
| Root docs URL configured in Olostep map node | `https://docs.tavily.com` |

