Generate personalized cold email icebreakers with Apify, Baserow and OpenRouter GPT-4.1

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-icebreakers-with-apify--baserow-and-openrouter-gpt-4-1-12778


# Generate personalized cold email icebreakers with Apify, Baserow and OpenRouter GPT-4.1

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate personalized cold email icebreakers with Apify, Baserow and OpenRouter GPT-4.1

**Purpose:**  
This workflow takes leads from a database (Baserow by default; Airtable nodes also included), scrapes each lead’s website (main page + selected internal pages), summarizes the site into a 3‑paragraph overview with an LLM, then generates a personalized cold-email **icebreaker** and **subject line** using another LLM. Finally, it updates the lead record with the generated content (Baserow update is the active output path; Airtable update nodes also exist).

**Primary use cases:**
- SDR/outbound personalization at scale (subject + first line).
- Automated website research and summarization.
- Flagging bad leads when websites fail/unreachable (“no content”).

### Logical Blocks
1.1 **Lead intake & iteration** (Manual trigger → fetch leads → loop/batching)  
1.2 **Scrape main URL & extract all links** (HTTP request → HTML link extraction → URL normalization)  
1.3 **Link filtering & page selection (max 5 URLs)** (internal-only filter → “desired pages” selection code)  
1.4 **Multi-page scraping + Apify fallback** (scrape selected pages normally; if none succeed, use Apify)  
1.5 **Content cleanup & aggregation** (HTML→Markdown → 5k trim → aggregate pages)  
1.6 **LLM: website overview generation**  
1.7 **LLM: icebreaker + subject generation (structured parsing)**  
1.8 **Database updates (success vs no-content/failure)**

> Important consistency note: prompts mention **Realtor/Real Estate** while database table labels/sticky notes mention **Med Spa**. Adjust prompts/keywords to match your niche.

---

## 2. Block-by-Block Analysis

### 1.1 Lead intake & iteration
**Overview:** Pulls leads needing an icebreaker from the database, reshapes fields, then processes them one by one via batching.

**Nodes involved:**
- When clicking ‘Execute workflow’ (Manual Trigger)
- Get Data from Sheet (Baserow)
- Get Sheet data (Airtable) *(present but not connected)*
- Person info (Set)
- Loop Over Items (Split in Batches)

#### Node: When clicking ‘Execute workflow’
- **Type/role:** Manual Trigger; entry point for the workflow.
- **Config:** No parameters.
- **Outputs:** Into **Get Data from Sheet**.
- **Failure modes:** None (manual start).

#### Node: Get Data from Sheet
- **Type/role:** Baserow node; fetch leads.
- **Config choices:**
  - **Database ID:** 350863
  - **Table ID:** 797289
  - **Limit:** 1 (processes one lead per run unless changed)
  - **Filter:** field `6809989` “not_empty” (exact meaning depends on your schema; likely “Icebreaker Created” or similar).
- **Outputs:** To **Person info**.
- **Credentials:** `Cognivio Acc` (Baserow API).
- **Failure modes:** invalid token, wrong table/database IDs, filter misconfiguration, empty result set.

#### Node: Get Sheet data *(unused)*
- **Type/role:** Airtable Search; alternative lead source.
- **Config choices:** searches Airtable where `{Icebreaker Created} = 'Under Review'`.
- **Not connected:** Doesn’t affect current run; keep only if you plan to switch input source.
- **Failure modes:** Airtable auth, base/table mismatch.

#### Node: Person info
- **Type/role:** Set node; normalizes lead fields for downstream expressions.
- **Fields created:** `id`, `First Name`, `Last Name`, `Title`, `Company Name`, `Website`.
- **Outputs:** to **Loop Over Items**.
- **Key expressions:** `={{ $json.Website }}`, etc.
- **Failure modes:** missing columns in input items → expressions resolve to empty.

#### Node: Loop Over Items
- **Type/role:** SplitInBatches; iterates over leads.
- **Config:** default options (batch size not specified; n8n default is typically 1).
- **Connections:**
  - Output continues to **Scrape Website** on the second output path (index 1).
  - Loop-back from DB update nodes returns here to process next item.
- **Failure modes:** incorrect wiring can skip items or create infinite loops.

---

### 1.2 Scrape main URL & extract links
**Overview:** Attempts to scrape the lead’s main website via direct HTTP. If direct scrape errors, an Apify fallback chain is available. The content is then parsed to extract all `<a href>` links.

**Nodes involved:**
- Scrape Website (HTTP Request)
- Apify Scraper (HTTP Request)
- Grab the Web URL (Set)
- Apify Scraper2 (HTTP Request)
- data or content both get forwarded (Set)
- Extract Links (HTML)
- Break the Array into Items (Split Out)

#### Node: Scrape Website
- **Type/role:** HTTP Request; fetches the main website.
- **Config choices:**
  - URL: `={{ $json.Website }}`
  - Timeout: 60s
  - Retries: max 5, wait 2s
  - **onError:** `continueErrorOutput` (errors go to error output)
- **Outputs:**
  - Main output → **data or content both get forwarded**
  - Error output → **Apify Scraper**
- **Failure modes:** DNS/SSL errors, blocked by bot protection, large pages, timeouts; continues due to error handling.

#### Node: Apify Scraper
- **Type/role:** HTTP Request; fallback scraper using Apify actor `mtrunkat~url-list-download-html`.
- **Config choices:**
  - URL contains placeholder: `token=YOUR_TOKEN_HERE<YOUR API KEY>`
  - JSON body uses `requestListSources: [{ url: "{{ $json.Website }}" }]`, `useChrome: true`, `useApifyProxy: true`.
  - **onError:** continueErrorOutput (wired both to scrape-set and another fallback chain)
- **Outputs:**
  - Main output → **Scraped from Apify**
  - Error output → **Grab the Web URL** (secondary fallback)
- **Failure modes:** missing/invalid Apify token, actor limits, proxy restrictions, actor changed response shape.

#### Node: Grab the Web URL
- **Type/role:** Set; attempts to reconstruct Website from debug fields.
- **Field:** `Website = {{ $json['#debug'].url }}{{ $json.Website }}`
- **Output:** to **Apify Scraper2**
- **Edge cases:** `#debug.url` often doesn’t exist; concatenation may produce malformed URL.

#### Node: Apify Scraper2
- **Type/role:** HTTP Request; alternate Apify actor `apify~website-content-crawler`.
- **Config:** similar token placeholder; sends URL `{{ $json.Website }}`.
- **Output:** to **Scraped from Apify**
- **Failure modes:** same as Apify Scraper; plus actor may return `page.content` instead of `fullHtml` (but later node expects `item.fullHtml`).

#### Node: data or content both get forwarded
- **Type/role:** Set; creates a unified `data` field from possible response shapes.
- **Field:** `data = {{ $json.page.content }}{{ $json.data }}`
- **Output:** to **Extract Links**
- **Edge cases:** if both are empty → `data` empty; if both exist → concatenated.

#### Node: Extract Links
- **Type/role:** HTML node; extracts all `<a>` href attributes into an array.
- **Config choices:**
  - Operation: extract HTML content
  - Selector: `a`, attribute `href`, return array
  - Clean-up + trim enabled
- **Output:** to **Break the Array into Items**
- **Edge cases:** non-HTML response, heavy JS sites without links in initial HTML.

#### Node: Break the Array into Items
- **Type/role:** Split Out; turns `links[]` into individual items.
- **Config:** fieldToSplitOut = `links`.
- **Output:** to **Check for Half URLs**
- **Edge cases:** missing `links` produces empty stream; but node is set to alwaysOutputData (so it will still pass data structure).

---

### 1.3 URL normalization, internal filtering, deduplication
**Overview:** Converts relative URLs to absolute, merges them back with already-absolute URLs, removes duplicates, then filters to internal URLs only.

**Nodes involved:**
- Check for Half URLs (If)
- Make Half URLs Full (Set)
- Combine Full and Empty URLs (Merge)
- Remove Duplicates (Code)
- Array into Items (Split Out)
- Filter out other URLs (Filter)

#### Node: Check for Half URLs
- **Type/role:** If; detects relative links (start with `/`).
- **Condition:** `links startsWith "/"`.
- **Outputs:**
  - True → **Make Half URLs Full**
  - False → **Combine Full and Empty URLs** (input 2)
- **Edge cases:** protocol-relative URLs `//example.com` not handled; relative links like `../about` not handled explicitly.

#### Node: Make Half URLs Full
- **Type/role:** Set; prepends the lead Website base to relative paths.
- **Expression:** removes trailing `/` from Website then appends link path.
- **Output:** to **Combine Full and Empty URLs**
- **Edge cases:** if Website itself includes path (not just domain), concatenation may be wrong.

#### Node: Combine Full and Empty URLs
- **Type/role:** Merge; combines the true/false streams of URLs.
- **Output:** to **Remove Duplicates**
- **Edge cases:** merge mode defaults matter; since no explicit mode is shown, verify it behaves as intended in your n8n version.

#### Node: Remove Duplicates
- **Type/role:** Code; deduplicates links and outputs a single combined item.
- **Key logic:**
  - Reads lead Website from `$('Person info').item.json.Website`.
  - Iterates items, takes `item.json.links`, trims, dedupes via `Set`.
  - Outputs `{ website, links }` as one item.
- **Output:** to **Array into Items**
- **Edge cases:** if `links` is not a string (e.g., object), `.trim()` fails; relies on extracted hrefs being strings.

#### Node: Array into Items
- **Type/role:** Split Out; splits combined `links[]` back into per-item entries.
- **Output:** to **Filter out other URLs**
- **Edge cases:** if the code node outputs `links` incorrectly, split fails.

#### Node: Filter out other URLs
- **Type/role:** Filter; keeps only internal links.
- **Conditions (OR):**
  1) link contains the exact Website string  
  2) link contains a variant where `www.` is injected after scheme (attempts to handle www/non-www)
- **Output:** to **Only Grab the Desired Pages**
- **Edge cases:**
  - If Website is like `example.com` without scheme, `contains` logic may miss matches.
  - Subdomains and canonical redirects may still slip through or be excluded incorrectly.

---

### 1.4 Page selection (max 5), scraping, and Apify fallback
**Overview:** From internal links, selects up to five “useful” pages (homepage always first) using keyword heuristics. Scrapes them via HTTP; if all scrapes fail, falls back to Apify.

**Nodes involved:**
- Only Grab the Desired Pages (Code)
- Array into Separate Items (Split Out)
- Scrape other pages (HTTP Request)
- Check for scrapes (Code)
- If there is Data (If)
- Normally Scraped (Set)
- Apify Scraper1 (HTTP Request)
- Scraped from Apify (Set)
- Merge All (Merge)

#### Node: Only Grab the Desired Pages
- **Type/role:** Code; selects “primary” pages using keywords and fills remaining slots using fallback logic.
- **Key config inside code:**
  - `keywords`: home/about/services/reviews/blog/contact, etc.
  - `blacklist`: privacy/terms/login/socials, etc.
  - `maxUrls = 5`
  - Normalizes relative URLs via `new URL(url, baseOrigin)`
  - Ensures homepage (original Website) is included first
  - Outputs: `{ id, firstName, urls: finalUrls }`
- **Output:** to **Array into Separate Items**
- **Edge cases:**
  - If Website is invalid URL string, normalization may fail and return only `{ url: originalUrl }` (note: that return shape differs from `{urls: [...]}`).
  - It lowercases URLs in `results` which can break case-sensitive paths (rare but possible).

#### Node: Array into Separate Items
- **Type/role:** Split Out; splits `urls[]` into individual items for scraping.
- **Output:** to **Scrape other pages**
- **Edge cases:** if previous node returned `{url: ...}` instead of `{urls: [...]}`, this node will output nothing.

#### Node: Scrape other pages
- **Type/role:** HTTP Request; scrapes each selected URL.
- **Config:**
  - URL: `={{ $json.urls }}`
  - Timeout: 70s
  - Response format: text
  - Batching: batchSize 1, interval 3000ms (politeness / rate limiting)
  - Retries: max 5, wait 2s
  - **onError:** continueRegularOutput
  - alwaysOutputData: true (ensures an item continues even on error)
- **Output:** to **Check for scrapes**
- **Edge cases:** returns different fields depending on HTTP node version and error conditions; can produce empty `data`.

#### Node: Check for scrapes
- **Type/role:** Code; decides whether at least one normal scrape produced usable data.
- **Logic:**
  - `dataItems = items.filter(item => item.json.data !== undefined && item.json.data !== null)`
  - `urlItems` logic is unusual: it references `$('Array into Separate Items').item.json.urls` inside the filter; this can behave unexpectedly because it’s not evaluating per item.
  - Outputs one summary item: `{ hasData, dataItems, urlItems }`
- **Output:** to **If there is Data**
- **Failure/edge cases:** if scrape node outputs HTML under a different property than `data`, `hasData` will be false even when content exists.

#### Node: If there is Data
- **Type/role:** If; routes based on `hasData`.
- **Condition:** `hasData is true`.
- **True output:** to **Normally Scraped**  
- **False output:** to **Apify Scraper1**
- **Edge cases:** boolean comparison is strict; ensure `hasData` is actual boolean.

#### Node: Normally Scraped
- **Type/role:** Set; prepares normal scrape content for downstream merge.
- **Field:** `data = {{ $('Check for scrapes').item.json.dataItems }}`
- **Output:** to **Merge All**
- **Edge cases:** `dataItems` is an array of items, not a string HTML. Downstream Markdown node expects string HTML; this can cause format mismatches unless n8n auto-stringifies.

#### Node: Apify Scraper1
- **Type/role:** HTTP Request; Apify fallback for selected URLs.
- **Config:** uses actor `mtrunkat~url-list-download-html` with URL `{{ $('Check for scrapes').item.json.urlItems }}`
- **Output:** to **Scraped from Apify**
- **Edge cases:** likely wrong: `urlItems` is an array of objects, not a single URL string. The Apify actor typically expects a list; but the body is configured as one `url`.

#### Node: Scraped from Apify
- **Type/role:** Set; maps Apify response to `data`.
- **Field:** `data = {{ $json.item.fullHtml }}`
- **Output:** to **Merge All** (input 2)
- **Edge cases:** if actor returns `html` or `page.content` instead of `item.fullHtml`, this will be empty.

#### Node: Merge All
- **Type/role:** Merge; combines “normally scraped” and “scraped from Apify” streams.
- **Output:** to **Converts HTML into easily readable format**
- **Edge cases:** merge mode again matters; validate that you end up with one item per scraped page.

---

### 1.5 Content cleanup & aggregation
**Overview:** Converts HTML to Markdown for token efficiency, trims each page to 5000 chars, then aggregates all page snippets into a single field used by the overview LLM.

**Nodes involved:**
- Converts HTML into easily readable format (Markdown)
- Limit Characters to 5k each (Set)
- Aggregate (Aggregate)

#### Node: Converts HTML into easily readable format
- **Type/role:** Markdown node; HTML → Markdown conversion.
- **Input:** `html = {{ $json.data }}`
- **Output:** to **Limit Characters to 5k each**
- **Failure modes:** invalid HTML, very large HTML can be slow; onError continueRegularOutput.

#### Node: Limit Characters to 5k each
- **Type/role:** Set; truncates content to max 5000 chars.
- **Expression:** `data = data.length > 5000 ? data.slice(0,5000) : data`
- **Output:** to **Aggregate**
- **Edge cases:** if `data` is not a string (array/object), `.length` semantics differ; `.slice` may fail.

#### Node: Aggregate
- **Type/role:** Aggregate node; collects all items into an array/combined structure.
- **Field aggregation:** aggregates `data` into output field `Webpage Overview`.
- **Output:** to **Website overview**
- **Edge cases:** if upstream produces no items, may output empty `Webpage Overview`.

---

### 1.6 LLM: Website overview generation
**Overview:** Uses OpenRouter GPT‑4.1 to produce a 3‑paragraph abstract from aggregated markdown.

**Nodes involved:**
- OpenRouter ThreeZero (OpenRouter Chat Model)
- Website overview (LangChain Chain LLM)

#### Node: OpenRouter ThreeZero
- **Type/role:** LangChain chat model provider for downstream LLM chain nodes.
- **Model:** `openai/gpt-4.1`
- **Credentials:** OpenRouter API credential `ThreeZero Digital Email Icebreaker`
- **Connections:** feeds as `ai_languageModel` to:
  - Website overview
  - Structured Output Parser
  - icebreaker & Subject
- **Failure modes:** invalid OpenRouter key, model unavailable, rate limits.

#### Node: Website overview
- **Type/role:** LangChain LLM chain; generates overview text.
- **Prompt:** explicitly instructs: Realtor/Real Estate niche, 3 paragraphs, spartan tone; return “no content” if empty/invalid.
- **Input variable:** `{{ $json['Webpage Overview'] }}`
- **Output:** goes to **Overview & Name**
- **Failure modes:** token limit (if aggregation too large), hallucination if scrape contains nav junk, or returns unexpected format.

---

### 1.7 LLM: Icebreaker + subject + structured parsing
**Overview:** Sets up the lead’s name and overview, generates an icebreaker and subject line with templates, then parses into structured fields.

**Nodes involved:**
- Overview & Name (Set)
- If it says no content (If)
- icebreaker & Subject (LangChain LLM chain)
- Structured Output Parser (Structured parser)
- Subject & Icebreaker (Set)
- Send icebreaker & subject no content (Set)

#### Node: Overview & Name
- **Type/role:** Set; prepares fields expected by the icebreaker LLM.
- **Fields:**
  - `Webpages Overview = {{ $json.text }}` (from Website overview output)
  - `First Name = {{ $('Loop Over Items').item.json['First Name'] }}`
- **Output:** to **If it says no content**
- **Edge cases:** name lookup depends on correct pairing; batching/paired items must remain consistent.

#### Node: If it says no content
- **Type/role:** If; intended to route “no content” vs “has content”.
- **Configured conditions (OR):**
  1) `Webpages Overview != "no content"`  
  2) `Webpages Overview contains "Website expired"`
- **Outputs:**
  - True → **icebreaker & Subject**
  - False → **Send icebreaker & subject no content**
- **Important logic issue:** as written, this routes to the LLM when overview is *not* “no content” (good), but also routes to LLM when it contains “Website expired” (which sounds like a failure case). If “Website expired” appears, you probably want the **no-content path** instead.

#### Node: icebreaker & Subject
- **Type/role:** LangChain LLM chain; generates the personalized icebreaker + subject.
- **Prompt specifics:**
  - Randomly pick one of two icebreaker templates.
  - Then choose one of three subject templates.
  - Outputs plain text; *but* node is configured with an output parser, implying structured JSON is expected.
  - Niche rule says Realtor/Real Estate (again mismatch if you’re targeting Med Spas).
- **Inputs:**
  - First Name: `{{ $json['First Name'] }}`
  - Webpages Overview: `{{ $json['Webpages Overview'] }}`
  - Company Name: `{{ $('Person info').item.json['Company Name'] }}`
- **Output:** to **Subject & Icebreaker** (after parser)
- **Failure modes:** model returns text not matching parser schema; prompt says “Plain Text only” but parser expects JSON-like structure.

#### Node: Structured Output Parser
- **Type/role:** LangChain structured parser; enforces output schema.
- **Schema example:**
  - `{ "Subject": "...", "Icebreaker": "..." }`
- **autoFix:** enabled (tries to repair malformed output)
- **Connected as:** `ai_outputParser` into **icebreaker & Subject**.
- **Edge cases:** autoFix can still fail on heavily non-JSON output; conflicts with “Plain Text only” instruction.

#### Node: Subject & Icebreaker
- **Type/role:** Set; maps parsed output to final fields.
- **Fields:**
  - `Subject Line = {{ $json.output.Subject }}`
  - `Icebreaker = {{ $json.output.Icebreaker }}`
- **Output:** to DB update node **Update Icebreaker and Subject**
- **Edge cases:** if parser outputs different keys (e.g., `subject` vs `Subject`), expressions resolve empty.

#### Node: Send icebreaker & subject no content
- **Type/role:** Set; fallback values when content is missing.
- **Fields set:** both Icebreaker and Subject Line become `{{ $json['Webpages Overview'] }}` (so likely “no content”).
- **Output:** to **Update NoContent**
- **Edge cases:** you may want more user-friendly placeholders than “no content”.

---

### 1.8 Database updates (success vs failure)
**Overview:** Writes generated fields back to Baserow. Airtable update nodes exist but are not connected in the active path.

**Nodes involved:**
- Update Icebreaker and Subject (Baserow)
- Update NoContent (Baserow)
- Icebreaker Update (Airtable) *(unused)*
- Update No content (Airtable) *(unused)*

#### Node: Update Icebreaker and Subject
- **Type/role:** Baserow Update Row.
- **Row ID:** `={{ $('Person info').item.json.id }}`
- **Fields written (by fieldId):**
  - 6809989 = Icebreaker
  - 6809970 = Subject Line
  - 6810068 = “Created”
  - 6810081 = Website overview text
- **Output:** loops back to **Loop Over Items**.
- **Failure modes:** wrong field IDs, row not found, auth errors, rate limits.

#### Node: Update NoContent
- **Type/role:** Baserow Update Row for failures.
- **Writes:**
  - Icebreaker/Subject from no-content node
  - 6810068 = “Failed”
  - 6810081 = Website overview text (may still be “no content”)
- **Output:** loops back to **Loop Over Items**.

#### Node: Icebreaker Update *(unused)*
- **Type/role:** Airtable update record using `id` as matching column.
- **Writes:** Icebreaker, Subject Line, Website Overview, sets “Icebreaker Created”.
- **Not connected:** doesn’t run.

#### Node: Update No content *(unused)*
- **Type/role:** Airtable update record for no-content path.
- **Not connected:** doesn’t run.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Entry point | — | Get Data from Sheet | ## Overview (workflow description, setup steps, use cases) |
| Get Data from Sheet | baserow | Fetch leads from Baserow | When clicking ‘Execute workflow’ | Person info | ## Get Leads from Database |
| Get Sheet data | airtable | (Alt) fetch leads from Airtable (unused) | — | — | ## Get Leads from Database |
| Person info | set | Normalize lead fields | Get Data from Sheet | Loop Over Items | ## Get Leads from Database |
| Loop Over Items | splitInBatches | Iterate through leads | Person info; Update Icebreaker and Subject; Update NoContent | Scrape Website | ## Get Leads from Database |
| Scrape Website | httpRequest | Scrape main website HTML | Loop Over Items | data or content both get forwarded; Apify Scraper | ## Scrape Main URL and get all the links on the page |
| Apify Scraper | httpRequest | Apify fallback scrape for main page | Scrape Website (error) | Scraped from Apify; Grab the Web URL | ## Scrape provided URL(s)… Retries using apify  /  ## To understand how to get the URL with embedded API |
| Grab the Web URL | set | Attempt to reconstruct Website URL | Apify Scraper (error) | Apify Scraper2 | ## Convert the half URL(s) into full |
| Apify Scraper2 | httpRequest | Alternative Apify crawler fallback | Grab the Web URL | Scraped from Apify | ## Scrape provided URL(s)… Retries using apify  /  ## To understand how to get the URL with embedded API |
| data or content both get forwarded | set | Normalize scraped content into `data` | Scrape Website | Extract Links | ## Scrape Main URL and get all the links on the page |
| Extract Links | html | Extract href links from HTML | data or content both get forwarded | Break the Array into Items | ## Scrape Main URL and get all the links on the page |
| Break the Array into Items | splitOut | One item per link | Extract Links | Check for Half URLs | ## Scrape Main URL and get all the links on the page |
| Check for Half URLs | if | Detect relative URLs | Break the Array into Items | Make Half URLs Full; Combine Full and Empty URLs | ## Convert the half URL(s) into full |
| Make Half URLs Full | set | Convert relative to absolute URLs | Check for Half URLs (true) | Combine Full and Empty URLs | ## Convert the half URL(s) into full |
| Combine Full and Empty URLs | merge | Merge absolute + converted URLs | Check for Half URLs (false); Make Half URLs Full | Remove Duplicates | ## Convert the half URL(s) into full |
| Remove Duplicates | code | Deduplicate link list | Combine Full and Empty URLs | Array into Items | ## Remove Duplicates |
| Array into Items | splitOut | Split deduped array back to items | Remove Duplicates | Filter out other URLs | ## Remove Duplicates / ## Remove Extras |
| Filter out other URLs | filter | Keep only internal URLs | Array into Items | Only Grab the Desired Pages | ## Remove External URLs / ## Remove Extras |
| Only Grab the Desired Pages | code | Select up to 5 useful pages | Filter out other URLs | Array into Separate Items | ## Detailed: Filters… IF IT WORKS DON'T TOUCH IT! / (detailed keywords note) |
| Array into Separate Items | splitOut | One item per selected URL | Only Grab the Desired Pages | Scrape other pages | ## Scrape provided URL(s)… Retries using apify |
| Scrape other pages | httpRequest | Scrape selected pages | Array into Separate Items | Check for scrapes | ## Scrape provided URL(s)… Retries using apify |
| Check for scrapes | code | Decide whether normal scrape succeeded | Scrape other pages | If there is Data | ## Passes Directly if… else uses the Apify scraper |
| If there is Data | if | Route: normal vs Apify fallback | Check for scrapes | Normally Scraped; Apify Scraper1 | ## Passes Directly if… else uses the Apify scraper |
| Normally Scraped | set | Prepare normal scraped content | If there is Data (true) | Merge All | ## Passes Directly if… else uses the Apify scraper |
| Apify Scraper1 | httpRequest | Apify fallback for selected pages | If there is Data (false) | Scraped from Apify | ## Scrape provided URL(s)… Retries using apify  /  ## To understand how… |
| Scraped from Apify | set | Map Apify HTML to `data` | Apify Scraper / Apify Scraper1 / Apify Scraper2 | Merge All | ## Scrape provided URL(s)… Retries using apify |
| Merge All | merge | Merge scrape streams | Normally Scraped; Scraped from Apify | Converts HTML into easily readable format | ## Scrape provided URL(s)… Retries using apify |
| Converts HTML into easily readable format | markdown | HTML→Markdown | Merge All | Limit Characters to 5k each | ## Shorten the HTML |
| Limit Characters to 5k each | set | Trim each page to 5000 chars | Converts HTML into easily readable format | Aggregate | ## Limit the material of the page(s) to 5000 characters |
| Aggregate | aggregate | Combine pages into `Webpage Overview` | Limit Characters to 5k each | Website overview | ## Scrape provided URL(s)… (mentions combining into one array) |
| OpenRouter ThreeZero | lmChatOpenRouter | LLM provider (GPT‑4.1 via OpenRouter) | — | Website overview / Structured Output Parser / icebreaker & Subject (ai ports) | ## LLM for generating Website Overview and Personalized Emails |
| Website overview | chainLlm | Generate 3‑paragraph site abstract | Aggregate | Overview & Name | ## LLM for generating Website Overview and Personalized Emails |
| Overview & Name | set | Prepare overview + first name | Website overview | If it says no content | ## LLM for generating Website Overview and Personalized Emails |
| If it says no content | if | Route success vs no-content | Overview & Name | icebreaker & Subject; Send icebreaker & subject no content | ## LLM for generating Website Overview and Personalized Emails |
| icebreaker & Subject | chainLlm | Generate icebreaker + subject | If it says no content (true) | Subject & Icebreaker | ## LLM for generating Website Overview and Personalized Emails |
| Structured Output Parser | outputParserStructured | Parse LLM output to {Subject, Icebreaker} | OpenRouter ThreeZero (ai) | icebreaker & Subject (ai parser link) | ## LLM for generating Website Overview and Personalized Emails |
| Subject & Icebreaker | set | Map parsed fields to DB-ready values | icebreaker & Subject | Update Icebreaker and Subject | ## Insert Icebreaker & Subject Line & Website Overview into the Database |
| Send icebreaker & subject no content | set | Fallback values on failure | If it says no content (false) | Update NoContent | ## Insert Icebreaker & Subject Line & Website Overview into the Database |
| Update Icebreaker and Subject | baserow | Update lead record (success) | Subject & Icebreaker | Loop Over Items | ## Insert Icebreaker & Subject Line & Website Overview into the Database |
| Update NoContent | baserow | Update lead record (failure) | Send icebreaker & subject no content | Loop Over Items | ## Insert Icebreaker & Subject Line & Website Overview into the Database |
| Icebreaker Update | airtable | (Alt) update Airtable (unused) | — | — | ## Insert Icebreaker & Subject Line & Website Overview into the Database |
| Update No content | airtable | (Alt) update Airtable failure path (unused) | — | — | ## Insert Icebreaker & Subject Line & Website Overview into the Database |
| Sticky Note | stickyNote | Comment | — | — | (content: ## Scrape Main URL and get all the links on the page) |
| Sticky Note1 | stickyNote | Comment | — | — | (content: ## Get Leads from Database) |
| Sticky Note2 | stickyNote | Comment | — | — | (content: ## Insert Icebreaker & Subject Line & Website Overview into the Database) |
| Sticky Note3 | stickyNote | Comment | — | — | (content: scraping/markdown/5k/aggregate/retries) |
| Sticky Note4 | stickyNote | Comment | — | — | (content: detailed page-selection guarantees + “IF IT WORKS DON'T TOUCH IT!”) |
| Sticky Note5 | stickyNote | Comment | — | — | (content: ## Convert the half URL(s) into full) |
| Sticky Note6 | stickyNote | Comment | — | — | (content: ## Remove External URLs) |
| Sticky Note7 | stickyNote | Comment | — | — | (content: normal scrape vs Apify fallback explanation) |
| Sticky Note8 | stickyNote | Comment | — | — | (content: full workflow overview + setup steps + use cases) |
| Sticky Note9 | stickyNote | Comment | — | — | (content: ## Remove Duplicates) |
| Sticky Note10 | stickyNote | Comment | — | — | (content: Apify actor URL parsing instructions) |
| Sticky Note11 | stickyNote | Comment | — | — | (content: ## Remove Extras) |
| Sticky Note12 | stickyNote | Comment | — | — | (content: shorten HTML/tokens) |
| Sticky Note13 | stickyNote | Comment | — | — | (content: desired pages keywords explanation) |
| Sticky Note14 | stickyNote | Comment | — | — | (content: limit to 5000 chars) |
| Sticky Note15 | stickyNote | Comment | — | — | (content: LLM nodes explanation) |

---

## 4. Reproducing the Workflow from Scratch (step-by-step)

1) **Create Trigger**
1. Add **Manual Trigger** node named **“When clicking ‘Execute workflow’”**.

2) **Add lead source (Baserow)**
2. Add **Baserow → List Rows** (or “Get Many”) node named **“Get Data from Sheet”**.
3. Configure credentials: create **Baserow API token** credential.
4. Set **Database ID = 350863**, **Table ID = 797289**.
5. Set **Limit = 1** (increase for more leads).
6. Add a filter matching your schema (in the JSON it filters field `6809989` with `not_empty`).

3) **Normalize lead fields**
7. Add **Set** node named **“Person info”**.
8. Create fields: `id`, `First Name`, `Last Name`, `Title`, `Company Name`, `Website` mapped from the Baserow columns.

4) **Batch processing**
9. Add **Split In Batches** named **“Loop Over Items”**.
10. Connect: Manual Trigger → Get Data from Sheet → Person info → Loop Over Items.

5) **Scrape main page (direct) with fallback**
11. Add **HTTP Request** named **“Scrape Website”**:
   - URL: `{{$json.Website}}`
   - Timeout: 60000ms
   - Retries: 5; wait: 2000ms
   - Error handling: **Continue (error output)**.
12. Connect Loop Over Items → Scrape Website.

6) **Normalize content and extract links**
13. Add **Set** node **“data or content both get forwarded”** with:
   - `data = {{$json.page.content}}{{$json.data}}`
14. Add **HTML** node **“Extract Links”**:
   - Extract `href` attributes from selector `a`
   - Return as array field `links`
15. Add **Split Out** node **“Break the Array into Items”** splitting `links`.
16. Wire: Scrape Website (main) → data or content both get forwarded → Extract Links → Break the Array into Items.

7) **Apify fallback for main scrape**
17. Add **HTTP Request** node **“Apify Scraper”**:
   - URL: `https://api.apify.com/v2/acts/mtrunkat~url-list-download-html/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`
   - Send JSON body containing `requestListSources: [{ url: {{$json.Website}} }]`, `useChrome: true`, `useApifyProxy: true`.
   - Error handling: Continue (error output).
18. Connect Scrape Website **error output** → Apify Scraper.
19. Add **Set** node **“Scraped from Apify”**:
   - `data = {{$json.item.fullHtml}}`
20. (Optional secondary fallback) Add **Set** “Grab the Web URL” and **HTTP Request** “Apify Scraper2” (actor `apify~website-content-crawler`) if you want the extra recovery path.

8) **Convert relative URLs + merge**
21. Add **If** node **“Check for Half URLs”**:
   - condition: `{{$json.links}} startsWith "/"`.
22. Add **Set** node **“Make Half URLs Full”**:
   - `links = {{ WebsiteWithoutTrailingSlash }} + {{$json.links}}`
23. Add **Merge** node **“Combine Full and Empty URLs”** and connect:
   - If(true) → Make Half URLs Full → Merge input 1
   - If(false) → Merge input 2

9) **Deduplicate & filter internal**
24. Add **Code** node **“Remove Duplicates”** implementing Set-based dedupe and output `{website, links:[...]}`.
25. Add **Split Out** node **“Array into Items”** splitting `links`.
26. Add **Filter** node **“Filter out other URLs”** to keep only internal URLs using “contains Website” rules.
27. Connect: Merge → Remove Duplicates → Array into Items → Filter out other URLs.

10) **Select up to 5 pages**
28. Add **Code** node **“Only Grab the Desired Pages”**:
   - Implement keyword/blacklist selection
   - Ensure output `{ id, firstName, urls: [...] }` and that homepage is included
29. Add **Split Out** node **“Array into Separate Items”** splitting `urls`.
30. Connect Filter out other URLs → Only Grab the Desired Pages → Array into Separate Items.

11) **Scrape selected pages + fallback**
31. Add **HTTP Request** node **“Scrape other pages”**:
   - URL: `{{$json.urls}}`
   - Response format: text
   - Timeout: 70000ms
   - Batching: size 1; interval 3000ms
   - Retries: 5; wait 2000ms
   - Continue on error; always output data
32. Add **Code** node **“Check for scrapes”** that sets `hasData` true if any item has usable HTML field.
33. Add **If** node **“If there is Data”** on `hasData`.
34. Add **Set** node **“Normally Scraped”** to map scraped content into `data`.
35. Add **HTTP Request** node **“Apify Scraper1”** (same Apify actor) for the fallback route.
36. Ensure **Scraped from Apify** maps the Apify HTML to `data`.
37. Add **Merge** node **“Merge All”** to unify both routes.

12) **Convert to Markdown, trim, and aggregate**
38. Add **Markdown** node **“Converts HTML into easily readable format”** with `html = {{$json.data}}`.
39. Add **Set** node **“Limit Characters to 5k each”** to truncate `data`.
40. Add **Aggregate** node **“Aggregate”** aggregating `data` into `Webpage Overview`.

13) **Configure OpenRouter model**
41. Add **OpenRouter Chat Model** node **“OpenRouter ThreeZero”**:
   - Model: `openai/gpt-4.1`
   - Configure OpenRouter API credential.

14) **LLM: Website overview**
42. Add **LangChain Chain LLM** node **“Website overview”**:
   - Prompt: 3-paragraph abstract, return “no content” if empty/invalid.
   - Human message uses `{{$json['Webpage Overview']}}`.
   - Attach language model: connect OpenRouter node to the chain’s **AI Language Model** input.

15) **Prepare fields + route no-content**
43. Add **Set** node **“Overview & Name”**:
   - `Webpages Overview = {{$json.text}}`
   - `First Name = {{$('Loop Over Items').item.json['First Name']}}`
44. Add **If** node **“If it says no content”**:
   - Decide whether to proceed to icebreaker generation or use fallback.

16) **LLM: Icebreaker + subject with structured parsing**
45. Add **Structured Output Parser** node with schema keys: `Subject`, `Icebreaker` and enable auto-fix.
46. Add **LangChain Chain LLM** node **“icebreaker & Subject”**:
   - Prompt with templates; ensure it outputs JSON matching parser (or change parser/prompt to align).
   - Attach language model and output parser via the OpenRouter node and the parser node.
47. Add **Set** node **“Subject & Icebreaker”** mapping:
   - `Subject Line = {{$json.output.Subject}}`
   - `Icebreaker = {{$json.output.Icebreaker}}`

17) **Fallback values for no-content**
48. Add **Set** node **“Send icebreaker & subject no content”**:
   - Set both fields to “no content” or a custom placeholder.

18) **Write back to Baserow**
49. Add **Baserow Update Row** node **“Update Icebreaker and Subject”**:
   - Row ID = `{{$('Person info').item.json.id}}`
   - Map your field IDs for Icebreaker, Subject, Status, Website Overview.
50. Add **Baserow Update Row** node **“Update NoContent”** similarly (status “Failed”).
51. Connect success update → **Loop Over Items** to continue the batch; connect failure update → **Loop Over Items**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview, setup steps, and use cases are included as a sticky note inside the canvas. | Sticky note “Overview” block (describes 1–9 steps and setup checklist). |
| Apify actor identification tip: copy the string after `/v2/acts/` and search it in Apify Store (example: `apify~website-content-crawler`). | Sticky note: “To understand how to get the URL with embedded API”. |
| Prompts currently instruct “Realtor/Real Estate websites” while tables/sticky notes reference “Med Spa”. Align prompts + page-selection keywords with your actual niche to avoid low-quality personalization. | Applies to “Website overview” and “icebreaker & Subject” prompts, and to the page-selection keyword list. |
| Structured parsing conflicts with “Output in Plain Text only” instruction in the icebreaker prompt. Either (a) change the prompt to output JSON, or (b) remove the structured parser and parse manually. | Applies to “icebreaker & Subject” + “Structured Output Parser”. |
| The “If it says no content” condition likely routes “Website expired” into the LLM path. Consider flipping that logic so expired sites go to the no-content update path. | Applies to routing before icebreaker generation. |