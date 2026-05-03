Generate and enrich local business leads with Decodo, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-and-enrich-local-business-leads-with-decodo--openai-and-gmail-14084


# Generate and enrich local business leads with Decodo, OpenAI and Gmail

# 1. Workflow Overview

This workflow automates a full local-business outbound pipeline: it discovers businesses on Google Maps, stores them in Google Sheets, enriches them by scraping their websites and analyzing them with OpenAI, then generates and sends personalized cold emails through Gmail to qualified leads.

It is designed for agencies, freelancers, B2B growth teams, and sales operators who want a repeatable lead generation and outreach system for local prospecting.

## 1.1 Block A — Trigger and Central Configuration

The workflow starts on a daily schedule and loads all operational parameters from a single Set node. These parameters control search terms, geography, sheet ID, lead cap, qualification threshold, and sender identity for outreach.

## 1.2 Block B — Lead Discovery from Google Maps

Using Decodo, the workflow scrapes Google Maps search results for a niche/city query. A Code node parses the returned markdown/text into structured lead records, then valid leads are written to Google Sheets.

## 1.3 Block C — Per-Lead Enrichment Routing

Each newly discovered lead is processed one by one. Leads without websites are marked and skipped; leads with websites are scraped for content and prepared for AI-based scoring and enrichment.

## 1.4 Block D — AI Enrichment and Lead Record Update

Website content is sent to OpenAI for qualification and enrichment. The AI response is parsed into structured fields such as score, summary, business type, pain points, and contact email, then written back to Google Sheets.

## 1.5 Block E — Outreach Candidate Retrieval and Qualification

Once discovery/enrichment processing progresses, the workflow loads all leads marked `Enriched` from Google Sheets and filters them against business rules: minimum score, valid website, available email, and not previously contacted.

## 1.6 Block F — Personalized Outreach Generation and Delivery

Qualified leads are processed one by one. Their websites are scraped again for more context, OpenAI writes a personalized cold email, email validity is checked, Gmail sends the message if valid, and the lead is marked as contacted in Google Sheets.

---

# 2. Block-by-Block Analysis

## Block A — Trigger and Central Configuration

### Overview
This block initializes the workflow. It defines when the automation runs and injects all reusable configuration values required by later nodes.

### Nodes Involved
- Schedule: Daily 10AM
- Search Config

### Node Details

#### 1. Schedule: Daily 10AM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Runs daily using cron expression `0 10 * * *`, meaning 10:00 every day.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Search Config`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Timezone mismatches if the n8n instance timezone is not what the operator expects.
- **Sub-workflow reference:** None.

#### 2. Search Config
- **Type and technical role:** `n8n-nodes-base.set`; central configuration store.
- **Configuration choices:** Defines:
  - `niche`: `digital marketing agency`
  - `city`: `New York`
  - `geo`: `United States`
  - `googleSheetId`: placeholder sheet ID
  - `maxLeads`: `15`
  - `scoreThreshold`: `6`
  - `senderName`: `Alex`
  - `senderCompany`: `GrowthLab Agency`
  - `senderOffer`: `helping local businesses grow their online presence`
- **Key expressions or variables used:** Later nodes reference these via expressions like `$('Search Config').first().json.googleSheetId`.
- **Input and output connections:** Input from `Schedule: Daily 10AM`; output to `Decodo: Scrape Google Maps`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** If `googleSheetId` remains a placeholder, all Google Sheets operations will fail. Incorrect `scoreThreshold` or `maxLeads` values may produce unexpected filtering behavior.
- **Sub-workflow reference:** None.

---

## Block B — Lead Discovery from Google Maps

### Overview
This block searches Google Maps for businesses matching the configured niche and city, parses the scraped content into structured lead records, validates them, and writes them to Google Sheets.

### Nodes Involved
- Decodo: Scrape Google Maps
- Parse Leads from Markdown
- Is Valid Lead?
- Save Lead to Google Sheets

### Node Details

#### 3. Decodo: Scrape Google Maps
- **Type and technical role:** `@decodo/n8n-nodes-decodo.decodo`; external scraping node for Google Maps results.
- **Configuration choices:**
  - `geo` comes from `{{$json.geo}}`
  - `url` is built dynamically as  
    `https://www.google.com/maps/search/<encoded niche + city>`
- **Key expressions or variables used:**
  - `={{ $json.geo }}`
  - `={{ 'https://www.google.com/maps/search/' + encodeURIComponent($json.niche + ' ' + $json.city) }}`
- **Input and output connections:** Input from `Search Config`; output to `Parse Leads from Markdown`.
- **Version-specific requirements:** Type version `1`. This is a Decodo community node and generally requires self-hosted n8n.
- **Edge cases or potential failure types:** Credential/auth issues with Decodo, scrape blocking, unexpected output format, empty results, timeout.
- **Sub-workflow reference:** None.

#### 4. Parse Leads from Markdown
- **Type and technical role:** `n8n-nodes-base.code`; transforms semi-structured Decodo output into structured lead items.
- **Configuration choices:**
  - Reads scraped content from `markdown`, `content`, `body`, `data`, or stringified JSON.
  - Pulls config from `Search Config`.
  - Applies regex-based extraction for phone, URL, rating, and address.
  - Caps results using `maxLeads`.
  - Creates lead objects with fields such as `id`, `businessName`, `address`, `phone`, `website`, `rating`, `status`, `aiScore`, `aiSummary`, `email`, `emailSent`, `scrapedAt`.
  - If nothing is parsed, emits one item with `error: 'No leads parsed'`.
- **Key expressions or variables used:**
  - `$('Search Config').first().json`
  - Generated ID: ``lead_${Date.now()}_${leadCount}``
- **Input and output connections:** Input from `Decodo: Scrape Google Maps`; output to `Is Valid Lead?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Parsing is heuristic and may misidentify business names or addresses.
  - Output format changes from Decodo could break regex extraction.
  - Duplicate IDs are unlikely but possible if multiple runs/processes overlap at millisecond precision.
  - If parsing fails, downstream validation may silently drop error records.
- **Sub-workflow reference:** None.

#### 5. Is Valid Lead?
- **Type and technical role:** `n8n-nodes-base.if`; filters parsed results.
- **Configuration choices:** Checks that `businessName` is not empty.
- **Key expressions or variables used:** `={{ $json.businessName }}`
- **Input and output connections:** Input from `Parse Leads from Markdown`; true output to `Save Lead to Google Sheets`. False branch is unused.
- **Version-specific requirements:** Type version `2.2`, condition engine version `2`.
- **Edge cases or potential failure types:** Records containing only the `error` field from the parser will fail this test and be discarded without logging.
- **Sub-workflow reference:** None.

#### 6. Save Lead to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`; persists discovered leads.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Sheet: `Leads`
  - Matching column: `id`
  - Writes columns:
    `id, city, email, niche, phone, rating, status, address, aiScore, website, aiSummary, emailSent, scrapedAt, businessName`
  - Document ID is read from `Search Config`.
- **Key expressions or variables used:** Multiple `{{$json...}}` mappings and `={{ $('Search Config').first().json.googleSheetId }}`
- **Input and output connections:** Input from `Is Valid Lead?`; output to `Split In Batches`.
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:** Invalid sheet ID, missing `Leads` tab, missing columns, OAuth expiration, rate limiting.
- **Sub-workflow reference:** None.

---

## Block C — Per-Lead Enrichment Routing

### Overview
This block iterates through newly saved leads and branches based on website availability. It prevents AI enrichment from running on leads that lack a website.

### Nodes Involved
- Split In Batches
- Has Website?
- Mark: No Website
- Update Lead (No Website)
- Decodo: Scrape Business Website
- Merge Website Content

### Node Details

#### 7. Split In Batches
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; processes leads sequentially/in batches.
- **Configuration choices:** Default options; batch size not explicitly customized.
- **Key expressions or variables used:** Later nodes reference current item via `$('Split In Batches').item.json...`
- **Input and output connections:**
  - Input from `Save Lead to Google Sheets`, `Update Lead (Enriched)`, and `Update Lead (No Website)`
  - First output goes to `Get Enriched Leads`
  - Loop output goes to `Has Website?`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:** Misunderstanding loop behavior can cause unexpected execution order. Large lead counts may increase runtime.
- **Sub-workflow reference:** None.

#### 8. Has Website?
- **Type and technical role:** `n8n-nodes-base.if`; branches based on website presence.
- **Configuration choices:** Checks that `website` is not empty.
- **Key expressions or variables used:** `={{ $json.website }}`
- **Input and output connections:** Input from `Split In Batches`; true output to `Decodo: Scrape Business Website`; false output to `Mark: No Website`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** A malformed or placeholder website string still counts as non-empty and proceeds to scraping.
- **Sub-workflow reference:** None.

#### 9. Mark: No Website
- **Type and technical role:** `n8n-nodes-base.set`; creates a minimal update payload for skipped leads.
- **Configuration choices:** Sets:
  - `id`
  - `status = 'No Website'`
  - `aiScore = 2`
  - `aiSummary = 'No website found — lead skipped from AI enrichment.'`
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:** Input from false branch of `Has Website?`; output to `Update Lead (No Website)`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Overwrites only selected fields; assumes `id` exists.
- **Sub-workflow reference:** None.

#### 10. Update Lead (No Website)
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates skipped leads in the sheet.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `id`
  - Updates `status`, `aiScore`, `aiSummary`
- **Key expressions or variables used:** `={{ $('Search Config').first().json.googleSheetId }}`
- **Input and output connections:** Input from `Mark: No Website`; output back to `Split In Batches`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** Matching row not found, sheet schema mismatch, credential errors.
- **Sub-workflow reference:** None.

#### 11. Decodo: Scrape Business Website
- **Type and technical role:** `@decodo/n8n-nodes-decodo.decodo`; scrapes website content for enrichment.
- **Configuration choices:**
  - Fixed `geo`: `United States`
  - `url` from `{{$json.website}}`
- **Key expressions or variables used:** `={{ $json.website }}`
- **Input and output connections:** Input from true branch of `Has Website?`; output to `Merge Website Content`.
- **Version-specific requirements:** Type version `1`; requires Decodo availability.
- **Edge cases or potential failure types:** Invalid URL, redirects, blocked scraping, long pages, timeout, binary/non-HTML pages.
- **Sub-workflow reference:** None.

#### 12. Merge Website Content
- **Type and technical role:** `n8n-nodes-base.set`; combines scraped website content with original lead data.
- **Configuration choices:**
  - Pulls source lead fields from `Split In Batches`
  - Adds `websiteContent` truncated to 3000 characters from scraped `markdown/content/body`
- **Key expressions or variables used:**
  - `={{ $('Split In Batches').item.json.id }}`
  - `={{ ($json.markdown || $json.content || $json.body || '').substring(0, 3000) }}`
- **Input and output connections:** Input from `Decodo: Scrape Business Website`; output to `AI: Score & Enrich Lead`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** If scraped content is empty, the AI node still runs with limited context.
- **Sub-workflow reference:** None.

---

## Block D — AI Enrichment and Lead Record Update

### Overview
This block asks OpenAI to score and enrich each website-backed lead, parses the model output, and updates the corresponding Google Sheets row.

### Nodes Involved
- AI: Score & Enrich Lead
- Parse AI Response
- Update Lead (Enriched)

### Node Details

#### 13. AI: Score & Enrich Lead
- **Type and technical role:** `n8n-nodes-base.openAi`; chat completion for business qualification.
- **Configuration choices:**
  - Resource: `chat`
  - `maxTokens`: `500`
  - `temperature`: `0.2`
- **Key expressions or variables used:** Prompt content is not visible in the JSON; only runtime options are defined here.
- **Input and output connections:** Input from `Merge Website Content`; output to `Parse AI Response`.
- **Version-specific requirements:** Type version `1.1`; requires OpenAI credentials.
- **Edge cases or potential failure types:** Missing prompt/instructions may result in poor or unusable output; token limits, auth failure, rate limits, malformed JSON replies.
- **Sub-workflow reference:** None.

#### 14. Parse AI Response
- **Type and technical role:** `n8n-nodes-base.code`; converts AI output into structured lead enrichment data.
- **Configuration choices:**
  - Reads content from common OpenAI response shapes.
  - Strips markdown code fences.
  - Attempts JSON parsing.
  - On failure, falls back to default values:
    - `aiScore = 5`
    - `aiSummary = 'AI parse failed: ...'`
    - unknown placeholders for business metadata
  - Emits a full lead record with status `Enriched`.
- **Key expressions or variables used:**
  - `$('Split In Batches').item.json`
  - `$json.message?.content || $json.choices?.[0]?.message?.content || $json.content || ''`
- **Input and output connections:** Input from `AI: Score & Enrich Lead`; output to `Update Lead (Enriched)`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If AI returns non-JSON or partial JSON, fallback data is used and may degrade downstream filtering quality.
- **Sub-workflow reference:** None.

#### 15. Update Lead (Enriched)
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates enriched fields in the sheet.
- **Configuration choices:**
  - Operation: `update`
  - Matching by `id`
  - Updates:
    `email, status, aiScore, aiSummary, painPoints, businessType, employeeCount, techSavviness`
- **Key expressions or variables used:** Field mappings from current JSON plus `Search Config` sheet ID.
- **Input and output connections:** Input from `Parse AI Response`; output back to `Split In Batches`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** Missing target columns in the sheet will break updates.
- **Sub-workflow reference:** None.

---

## Block E — Outreach Candidate Retrieval and Qualification

### Overview
This block retrieves all leads marked as enriched and applies business rules to decide which leads are eligible for outreach.

### Nodes Involved
- Get Enriched Leads
- Filter by Score Threshold
- Qualified Leads Exist?
- Split In Batches1

### Node Details

#### 16. Get Enriched Leads
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads candidate leads from the sheet.
- **Configuration choices:**
  - Filters rows where `status = Enriched`
  - Executes once
  - Sheet name: `Leads`
- **Key expressions or variables used:** `={{ $('Search Config').first().json.googleSheetId }}`
- **Input and output connections:** Input from `Split In Batches`; output to `Filter by Score Threshold`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** If no enriched rows exist, downstream logic receives an empty dataset or no items depending on Sheets behavior.
- **Sub-workflow reference:** None.

#### 17. Filter by Score Threshold
- **Type and technical role:** `n8n-nodes-base.code`; applies qualification rules.
- **Configuration choices:**
  - Reads `scoreThreshold` from `Search Config`
  - Keeps only leads where:
    - `aiScore >= threshold`
    - `email` exists and length > 3
    - `website` exists and length > 5
    - `emailSent !== 'Yes'`
  - If none qualify, returns one placeholder item with `__noLeads: true`
- **Key expressions or variables used:** `$('Search Config').first().json`
- **Input and output connections:** Input from `Get Enriched Leads`; output to `Qualified Leads Exist?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If `aiScore` is stored as text in unexpected format, numeric conversion may reduce it to `0`.
- **Sub-workflow reference:** None.

#### 18. Qualified Leads Exist?
- **Type and technical role:** `n8n-nodes-base.if`; prevents outreach execution when placeholder “no leads” item is present.
- **Configuration choices:** Proceeds only when `__noLeads` is not equal to `true`.
- **Key expressions or variables used:** `={{ $json.__noLeads }}`
- **Input and output connections:** Input from `Filter by Score Threshold`; true output to `Split In Batches1`. False branch unused.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** If a real lead ever contains `__noLeads: true`, it would be incorrectly filtered out.
- **Sub-workflow reference:** None.

#### 19. Split In Batches1
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through outreach-qualified leads.
- **Configuration choices:** Default options.
- **Key expressions or variables used:** Referenced by later nodes through `$('Split In Batches1').item.json...`
- **Input and output connections:**
  - Input from `Qualified Leads Exist?`, `Mark as Contacted`, and `Mark: Invalid Email`
  - Loop path sends items to `Decodo: Deep Scrape Website`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:** Same looping caveats as the first Split in Batches node.
- **Sub-workflow reference:** None.

---

## Block F — Personalized Outreach Generation and Delivery

### Overview
This block deep-scrapes each qualified lead’s website, uses OpenAI to generate a personalized email, validates the recipient email, sends it via Gmail, and updates contact status.

### Nodes Involved
- Decodo: Deep Scrape Website
- Merge Lead + Content
- AI: Write Cold Email
- Parse Email Content
- Valid Email Address?
- Gmail: Send Cold Email
- Mark: Invalid Email
- Mark as Contacted

### Node Details

#### 20. Decodo: Deep Scrape Website
- **Type and technical role:** `@decodo/n8n-nodes-decodo.decodo`; re-scrapes the website for richer outreach context.
- **Configuration choices:**
  - Fixed `geo`: `United States`
  - URL from `{{$json.website}}`
- **Key expressions or variables used:** `={{ $json.website }}`
- **Input and output connections:** Input from `Split In Batches1`; output to `Merge Lead + Content`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Same as prior Decodo website scrape node.
- **Sub-workflow reference:** None.

#### 21. Merge Lead + Content
- **Type and technical role:** `n8n-nodes-base.set`; joins original lead data, sender config, and scraped website content.
- **Configuration choices:** Maps:
  - lead identity and contact fields
  - `aiSummary`, `painPoints`
  - sender fields from `Search Config`
  - `websiteContent` truncated to 3500 characters
- **Key expressions or variables used:**
  - `$('Split In Batches1').item.json...`
  - `$('Search Config').first().json.senderName`
  - `={{ ($json.markdown || $json.content || $json.body || '').substring(0, 3500) }}`
- **Input and output connections:** Input from `Decodo: Deep Scrape Website`; output to `AI: Write Cold Email`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Missing scraped content reduces personalization quality.
- **Sub-workflow reference:** None.

#### 22. AI: Write Cold Email
- **Type and technical role:** `n8n-nodes-base.openAi`; generates outreach copy.
- **Configuration choices:**
  - Resource: `chat`
  - `maxTokens`: `600`
  - `temperature`: `0.7`
- **Key expressions or variables used:** Prompt body is not visible in the exported JSON.
- **Input and output connections:** Input from `Merge Lead + Content`; output to `Parse Email Content`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Invalid JSON response, overly long or off-brand content, auth/rate-limit issues.
- **Sub-workflow reference:** None.

#### 23. Parse Email Content
- **Type and technical role:** `n8n-nodes-base.code`; parses AI email output and applies fallbacks.
- **Configuration choices:**
  - Reads AI text from common OpenAI response structures.
  - Attempts JSON parsing after removing code fences.
  - Fallback email subject/body is generated if parsing fails.
  - Ensures minimum length for subject and HTML body.
  - Outputs normalized fields:
    `id, businessName, recipientEmail, website, emailSubject, emailBodyHtml, senderName, senderCompany`
- **Key expressions or variables used:**
  - `$('Merge Lead + Content').item.json`
  - Fallback templates referencing `lead.businessName`, `lead.senderOffer`, `lead.senderName`, `lead.senderCompany`
- **Input and output connections:** Input from `AI: Write Cold Email`; output to `Valid Email Address?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If AI returns plain text instead of JSON, fallback content is used. If recipient email is empty, later validation blocks sending.
- **Sub-workflow reference:** None.

#### 24. Valid Email Address?
- **Type and technical role:** `n8n-nodes-base.if`; basic email validation gate.
- **Configuration choices:** Checks whether `recipientEmail` contains `@`.
- **Key expressions or variables used:** `={{ $json.recipientEmail }}`
- **Input and output connections:** Input from `Parse Email Content`; true output to `Gmail: Send Cold Email`; false output to `Mark: Invalid Email`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** This is a minimal validation rule; malformed addresses like `a@` or `@test` will still pass.
- **Sub-workflow reference:** None.

#### 25. Gmail: Send Cold Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends outbound email.
- **Configuration choices:**
  - `sendTo`: recipient email
  - `subject`: generated email subject
  - `message`: generated HTML body
- **Key expressions or variables used:**
  - `={{ $json.recipientEmail }}`
  - `={{ $json.emailSubject }}`
  - `={{ $json.emailBodyHtml }}`
- **Input and output connections:** Input from true branch of `Valid Email Address?`; output to `Mark as Contacted`.
- **Version-specific requirements:** Type version `2.1`; requires Gmail OAuth2.
- **Edge cases or potential failure types:** Gmail auth issues, sending limits, spam controls, HTML rendering quirks.
- **Sub-workflow reference:** None.

#### 26. Mark: Invalid Email
- **Type and technical role:** `n8n-nodes-base.googleSheets`; records leads that lack a valid email.
- **Configuration choices:**
  - Operation: `update`
  - Matching by `id`
  - Sets `status = 'No Email'`
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:** Input from false branch of `Valid Email Address?`; output back to `Split In Batches1`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** If the row cannot be matched by `id`, status will not update.
- **Sub-workflow reference:** None.

#### 27. Mark as Contacted
- **Type and technical role:** `n8n-nodes-base.googleSheets`; records successful outreach.
- **Configuration choices:**
  - Operation: `update`
  - Matching by `id`
  - Sets:
    - `status = 'Contacted'`
    - `emailSent = 'Yes'`
    - `contactedAt = current ISO timestamp`
    - `emailSubject = sent subject`
- **Key expressions or variables used:**
  - `={{ $('Parse Email Content').item.json.id }}`
  - `={{ new Date().toISOString() }}`
  - `={{ $('Parse Email Content').item.json.emailSubject }}`
- **Input and output connections:** Input from `Gmail: Send Cold Email`; output back to `Split In Batches1`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** If Gmail succeeds but Sheets update fails, the email is sent but the lead is not marked contacted, creating duplicate-send risk in future runs.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule: Daily 10AM | Schedule Trigger | Starts the workflow every day at 10:00 |  | Search Config | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| Search Config | Set | Defines all reusable search, scoring, sheet, and sender settings | Schedule: Daily 10AM | Decodo: Scrape Google Maps | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| Decodo: Scrape Google Maps | Decodo | Scrapes Google Maps search results for businesses | Search Config | Parse Leads from Markdown | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| Parse Leads from Markdown | Code | Converts scraped map content into structured lead records | Decodo: Scrape Google Maps | Is Valid Lead? | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| Is Valid Lead? | If | Filters out records without a business name | Parse Leads from Markdown | Save Lead to Google Sheets | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| Save Lead to Google Sheets | Google Sheets | Appends or updates discovered leads in the Leads sheet | Is Valid Lead? | Split In Batches | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| Split In Batches | Split In Batches | Iterates through discovered leads for enrichment | Save Lead to Google Sheets; Update Lead (Enriched); Update Lead (No Website) | Get Enriched Leads; Has Website? | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Has Website? | If | Branches leads based on website availability | Split In Batches | Decodo: Scrape Business Website; Mark: No Website | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Mark: No Website | Set | Prepares skipped-lead update payload | Has Website? | Update Lead (No Website) | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Update Lead (No Website) | Google Sheets | Marks leads without websites in Google Sheets | Mark: No Website | Split In Batches | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Decodo: Scrape Business Website | Decodo | Scrapes the lead’s website for enrichment context | Has Website? | Merge Website Content | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Merge Website Content | Set | Combines scraped website text with original lead fields | Decodo: Scrape Business Website | AI: Score & Enrich Lead | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| AI: Score & Enrich Lead | OpenAI | Scores and enriches each lead using website content | Merge Website Content | Parse AI Response | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Parse AI Response | Code | Parses OpenAI JSON output into normalized lead fields | AI: Score & Enrich Lead | Update Lead (Enriched) | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Update Lead (Enriched) | Google Sheets | Writes enrichment results back to Google Sheets | Parse AI Response | Split In Batches | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Get Enriched Leads | Google Sheets | Loads all enriched leads from the sheet | Split In Batches | Filter by Score Threshold | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Filter by Score Threshold | Code | Filters enriched leads for outreach eligibility | Get Enriched Leads | Qualified Leads Exist? | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Qualified Leads Exist? | If | Stops outreach if no qualified lead exists | Filter by Score Threshold | Split In Batches1 | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| Split In Batches1 | Split In Batches | Iterates through qualified leads for outreach | Qualified Leads Exist?; Mark as Contacted; Mark: Invalid Email | Decodo: Deep Scrape Website | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Decodo: Deep Scrape Website | Decodo | Scrapes website content again for email personalization | Split In Batches1 | Merge Lead + Content | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Merge Lead + Content | Set | Merges lead data, sender config, and website content for email writing | Decodo: Deep Scrape Website | AI: Write Cold Email | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| AI: Write Cold Email | OpenAI | Generates personalized cold email content | Merge Lead + Content | Parse Email Content | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Parse Email Content | Code | Parses AI email JSON and applies fallback email templates | AI: Write Cold Email | Valid Email Address? | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Valid Email Address? | If | Checks whether the recipient email looks usable | Parse Email Content | Gmail: Send Cold Email; Mark: Invalid Email | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Gmail: Send Cold Email | Gmail | Sends the personalized outreach email | Valid Email Address? | Mark as Contacted | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Mark: Invalid Email | Google Sheets | Marks leads with invalid/missing email as No Email | Valid Email Address? | Split In Batches1 | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| Mark as Contacted | Google Sheets | Marks successfully emailed leads as contacted | Gmail: Send Cold Email | Split In Batches1 | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |
| 📋 WF1 Setup Guide | Sticky Note | Setup and operating notes |  |  | ## End-to-End Local Business Lead Generation, Enrichment and Outreach Pipeline with Decodo\n\nThis workflow finds local business leads on Google Maps, enriches them with website and AI analysis, and sends personalized cold emails to qualified prospects automatically.\n\n## Who’s it for?\nThis template is for agencies, freelancers, B2B growth teams, and sales operators who want a repeatable outbound lead generation system for local business prospecting.\n\n## How it works\n1. The workflow starts on a schedule and loads the search, scoring, and sender settings from one configuration node.\n2. Decodo scrapes Google Maps search results for businesses that match your niche and city.\n3. The workflow extracts lead details such as business name, address, phone number, website, and rating, then stores each lead in Google Sheets.\n4. Each discovered lead is checked for a website. If no website is available, the lead is marked and skipped from enrichment.\n5. For leads with a website, Decodo scrapes the site and OpenAI scores the business, summarizes it, and suggests enrichment fields such as business type, pain points, and contact email.\n6. The workflow updates the lead in Google Sheets with the enrichment data.\n7. Enriched leads are filtered by score threshold, website presence, email availability, and contact status.\n8. For each qualified lead, the workflow scrapes more website content and uses OpenAI to write a personalized outreach email.\n9. Gmail sends the email, and the lead is marked as contacted in Google Sheets.\n\n## How to set up\nAdd credentials for Decodo, OpenAI, Google Sheets, and Gmail. Replace the placeholder Google Sheet ID in the configuration node. Create a `Leads` sheet with all required columns used in the workflow. Update the niche, city, geo, max leads, score threshold, and sender fields. This template uses the Decodo community node, so it is intended for self-hosted n8n only. Replace the placeholder image above with a real workflow screenshot before submission. |
| 📍 Stage 1: Discovery | Sticky Note | Visual stage label for discovery block |  |  | ## 📍 Stage 1: Discovery\n\nScrapes Google Maps for business leads and saves them to Google Sheets with status='New' |
| 📍 Stage 2: Enrichment | Sticky Note | Visual stage label for enrichment block |  |  | ## 📍 Stage 2: Enrichment\n\nProcesses 'New' leads, scrapes websites, uses AI to score and enrich, updates status='Enriched' |
| 📍 Stage 3: Outreach | Sticky Note | Visual stage label for outreach block |  |  | ## 📍 Stage 3: Outreach\n\nFilters enriched leads by score, generates personalized emails with AI, sends via Gmail, marks as 'Contacted' |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Generate and enrich local business leads with Decodo, OpenAI and Gmail`.

2. **Add a Schedule Trigger node**
   - Type: `Schedule Trigger`
   - Name: `Schedule: Daily 10AM`
   - Set cron expression to: `0 10 * * *`
   - Confirm the instance timezone is correct.

3. **Add a Set node for central configuration**
   - Type: `Set`
   - Name: `Search Config`
   - Create these fields:
     - `niche` = `digital marketing agency`
     - `city` = `New York`
     - `geo` = `United States`
     - `googleSheetId` = your Google Sheet ID
     - `maxLeads` = `15`
     - `scoreThreshold` = `6`
     - `senderName` = `Alex`
     - `senderCompany` = `GrowthLab Agency`
     - `senderOffer` = `helping local businesses grow their online presence`
   - Connect `Schedule: Daily 10AM` → `Search Config`.

4. **Add a Decodo node for Google Maps scraping**
   - Type: `Decodo` community node
   - Name: `Decodo: Scrape Google Maps`
   - Set `geo` to: `={{ $json.geo }}`
   - Set `url` to:  
     `={{ 'https://www.google.com/maps/search/' + encodeURIComponent($json.niche + ' ' + $json.city) }}`
   - Configure Decodo credentials.
   - Connect `Search Config` → `Decodo: Scrape Google Maps`.

5. **Add a Code node to parse scraped map output**
   - Type: `Code`
   - Name: `Parse Leads from Markdown`
   - Paste the parsing code from the workflow JSON.
   - Ensure mode supports returning multiple items.
   - Connect `Decodo: Scrape Google Maps` → `Parse Leads from Markdown`.

6. **Add an If node to validate parsed leads**
   - Type: `If`
   - Name: `Is Valid Lead?`
   - Condition: string `businessName` is not empty
   - Expression: `={{ $json.businessName }}`
   - Connect `Parse Leads from Markdown` → `Is Valid Lead?`.

7. **Prepare Google Sheets**
   - Create a spreadsheet.
   - Create a sheet named `Leads`.
   - Add at least these columns:
     - `id`
     - `businessName`
     - `address`
     - `phone`
     - `website`
     - `rating`
     - `niche`
     - `city`
     - `status`
     - `aiScore`
     - `aiSummary`
     - `employeeCount`
     - `businessType`
     - `techSavviness`
     - `painPoints`
     - `email`
     - `emailSent`
     - `scrapedAt`
     - `contactedAt`
     - `emailSubject`

8. **Add a Google Sheets node to save discovered leads**
   - Type: `Google Sheets`
   - Name: `Save Lead to Google Sheets`
   - Credential: Google Sheets OAuth2
   - Operation: `Append or Update`
   - Sheet: `Leads`
   - Document ID: `={{ $('Search Config').first().json.googleSheetId }}`
   - Matching column: `id`
   - Map these fields:
     - `id`, `city`, `email`, `niche`, `phone`, `rating`, `status`, `address`, `aiScore`, `website`, `aiSummary`, `emailSent`, `scrapedAt`, `businessName`
   - Connect `Is Valid Lead?` true branch → `Save Lead to Google Sheets`.

9. **Add a Split In Batches node for enrichment iteration**
   - Type: `Split In Batches`
   - Name: `Split In Batches`
   - Leave default options unless you want to control batch size.
   - Connect `Save Lead to Google Sheets` → `Split In Batches`.

10. **Add an If node to check website presence**
    - Type: `If`
    - Name: `Has Website?`
    - Condition: string `website` is not empty
    - Expression: `={{ $json.website }}`
    - Connect `Split In Batches` loop output → `Has Website?`.

11. **Add a Set node for leads without websites**
    - Type: `Set`
    - Name: `Mark: No Website`
    - Set:
      - `id` = `={{ $json.id }}`
      - `status` = `No Website`
      - `aiScore` = `2`
      - `aiSummary` = `No website found — lead skipped from AI enrichment.`
    - Connect `Has Website?` false branch → `Mark: No Website`.

12. **Add a Google Sheets update node for no-website leads**
    - Type: `Google Sheets`
    - Name: `Update Lead (No Website)`
    - Operation: `Update`
    - Sheet: `Leads`
    - Document ID from `Search Config`
    - Matching column: `id`
    - Update: `id`, `status`, `aiScore`, `aiSummary`
    - Connect `Mark: No Website` → `Update Lead (No Website)`.

13. **Loop no-website leads back to the batch node**
    - Connect `Update Lead (No Website)` → `Split In Batches`.

14. **Add a Decodo node to scrape each business website**
    - Type: `Decodo`
    - Name: `Decodo: Scrape Business Website`
    - `geo` = `United States`
    - `url` = `={{ $json.website }}`
    - Connect `Has Website?` true branch → `Decodo: Scrape Business Website`.

15. **Add a Set node to combine lead data and scraped content**
    - Type: `Set`
    - Name: `Merge Website Content`
    - Add fields:
      - `id` = `={{ $('Split In Batches').item.json.id }}`
      - `businessName` = `={{ $('Split In Batches').item.json.businessName }}`
      - `website` = `={{ $('Split In Batches').item.json.website }}`
      - `niche` = `={{ $('Split In Batches').item.json.niche }}`
      - `city` = `={{ $('Split In Batches').item.json.city }}`
      - `websiteContent` = `={{ ($json.markdown || $json.content || $json.body || '').substring(0, 3000) }}`
    - Connect `Decodo: Scrape Business Website` → `Merge Website Content`.

16. **Add an OpenAI node for scoring/enrichment**
    - Type: `OpenAI`
    - Name: `AI: Score & Enrich Lead`
    - Resource: `Chat`
    - Set:
      - `temperature` = `0.2`
      - `maxTokens` = `500`
    - Configure OpenAI credentials.
    - Add your chat prompt so the model returns JSON with fields such as:
      - `aiScore`
      - `aiSummary`
      - `employeeCount`
      - `businessType`
      - `techSavviness`
      - `painPoints`
      - `email`
    - Important: the exported JSON does not include the actual prompt text, so you must define it manually.
    - Connect `Merge Website Content` → `AI: Score & Enrich Lead`.

17. **Add a Code node to parse the AI enrichment output**
    - Type: `Code`
    - Name: `Parse AI Response`
    - Mode: `Run once for each item`
    - Paste the code from the workflow JSON.
    - Connect `AI: Score & Enrich Lead` → `Parse AI Response`.

18. **Add a Google Sheets node to save enrichment results**
    - Type: `Google Sheets`
    - Name: `Update Lead (Enriched)`
    - Operation: `Update`
    - Sheet: `Leads`
    - Match by `id`
    - Update fields:
      - `id`
      - `email`
      - `status`
      - `aiScore`
      - `aiSummary`
      - `painPoints`
      - `businessType`
      - `employeeCount`
      - `techSavviness`
    - Connect `Parse AI Response` → `Update Lead (Enriched)`.

19. **Loop enriched leads back to the enrichment iterator**
    - Connect `Update Lead (Enriched)` → `Split In Batches`.

20. **Add a Google Sheets node to retrieve enriched leads**
    - Type: `Google Sheets`
    - Name: `Get Enriched Leads`
    - Operation: read rows with filter
    - Filter where `status = Enriched`
    - Set `Execute Once = true`
    - Connect the first output of `Split In Batches` → `Get Enriched Leads`.

21. **Add a Code node for outreach qualification**
    - Type: `Code`
    - Name: `Filter by Score Threshold`
    - Paste the code from the workflow JSON.
    - This code filters for:
      - score above threshold
      - non-empty email
      - non-empty website
      - `emailSent !== 'Yes'`
    - Connect `Get Enriched Leads` → `Filter by Score Threshold`.

22. **Add an If node to stop outreach when no qualified leads exist**
    - Type: `If`
    - Name: `Qualified Leads Exist?`
    - Condition: `={{ $json.__noLeads }}` is not equal to `true`
    - Connect `Filter by Score Threshold` → `Qualified Leads Exist?`.

23. **Add a second Split In Batches node for outreach**
    - Type: `Split In Batches`
    - Name: `Split In Batches1`
    - Connect `Qualified Leads Exist?` true branch → `Split In Batches1`.

24. **Add a Decodo node for deeper website scraping**
    - Type: `Decodo`
    - Name: `Decodo: Deep Scrape Website`
    - `geo` = `United States`
    - `url` = `={{ $json.website }}`
    - Connect `Split In Batches1` loop output → `Decodo: Deep Scrape Website`.

25. **Add a Set node to merge lead, sender, and website content**
    - Type: `Set`
    - Name: `Merge Lead + Content`
    - Add fields:
      - `id` = `={{ $('Split In Batches1').item.json.id }}`
      - `businessName` = `={{ $('Split In Batches1').item.json.businessName }}`
      - `recipientEmail` = `={{ $('Split In Batches1').item.json.email }}`
      - `website` = `={{ $('Split In Batches1').item.json.website }}`
      - `aiSummary` = `={{ $('Split In Batches1').item.json.aiSummary }}`
      - `painPoints` = `={{ $('Split In Batches1').item.json.painPoints }}`
      - `niche` = `={{ $('Split In Batches1').item.json.niche }}`
      - `city` = `={{ $('Split In Batches1').item.json.city }}`
      - `senderName` = `={{ $('Search Config').first().json.senderName }}`
      - `senderCompany` = `={{ $('Search Config').first().json.senderCompany }}`
      - `senderOffer` = `={{ $('Search Config').first().json.senderOffer }}`
      - `websiteContent` = `={{ ($json.markdown || $json.content || $json.body || '').substring(0, 3500) }}`
    - Connect `Decodo: Deep Scrape Website` → `Merge Lead + Content`.

26. **Add an OpenAI node to generate outreach emails**
    - Type: `OpenAI`
    - Name: `AI: Write Cold Email`
    - Resource: `Chat`
    - Set:
      - `temperature` = `0.7`
      - `maxTokens` = `600`
    - Add your prompt so the model returns JSON with:
      - `emailSubject`
      - `emailBodyHtml`
    - The original exported JSON does not contain the prompt text, so define it manually.
    - Connect `Merge Lead + Content` → `AI: Write Cold Email`.

27. **Add a Code node to parse the AI email output**
    - Type: `Code`
    - Name: `Parse Email Content`
    - Mode: `Run once for each item`
    - Paste the code from the workflow JSON.
    - Connect `AI: Write Cold Email` → `Parse Email Content`.

28. **Add an If node to validate the recipient email**
    - Type: `If`
    - Name: `Valid Email Address?`
    - Condition: `recipientEmail` contains `@`
    - Expression: `={{ $json.recipientEmail }}`
    - Connect `Parse Email Content` → `Valid Email Address?`.

29. **Add a Gmail node to send the email**
    - Type: `Gmail`
    - Name: `Gmail: Send Cold Email`
    - Credential: Gmail OAuth2
    - Set:
      - `To` = `={{ $json.recipientEmail }}`
      - `Subject` = `={{ $json.emailSubject }}`
      - `Message` = `={{ $json.emailBodyHtml }}`
    - Connect `Valid Email Address?` true branch → `Gmail: Send Cold Email`.

30. **Add a Google Sheets node to mark invalid-email leads**
    - Type: `Google Sheets`
    - Name: `Mark: Invalid Email`
    - Operation: `Update`
    - Match by `id`
    - Set `status = No Email`
    - Connect `Valid Email Address?` false branch → `Mark: Invalid Email`.

31. **Loop invalid-email leads back to outreach iterator**
    - Connect `Mark: Invalid Email` → `Split In Batches1`.

32. **Add a Google Sheets node to mark successful contacts**
    - Type: `Google Sheets`
    - Name: `Mark as Contacted`
    - Operation: `Update`
    - Match by `id`
    - Update:
      - `id` = `={{ $('Parse Email Content').item.json.id }}`
      - `status` = `Contacted`
      - `emailSent` = `Yes`
      - `contactedAt` = `={{ new Date().toISOString() }}`
      - `emailSubject` = `={{ $('Parse Email Content').item.json.emailSubject }}`
    - Connect `Gmail: Send Cold Email` → `Mark as Contacted`.

33. **Loop contacted leads back to outreach iterator**
    - Connect `Mark as Contacted` → `Split In Batches1`.

34. **Add sticky notes if desired**
    - One setup note explaining credentials, required sheet columns, and Decodo self-hosting requirement.
    - One note for each stage:
      - Discovery
      - Enrichment
      - Outreach

35. **Configure credentials**
    - **Decodo:** required for all scrape nodes; community node likely self-hosted only.
    - **Google Sheets OAuth2:** required for save, update, and retrieval nodes.
    - **OpenAI:** required for both AI nodes.
    - **Gmail OAuth2:** required for outbound emails.

36. **Test in phases**
    - First test trigger → discovery only.
    - Then test enrichment with a few leads.
    - Then test outreach with a safe recipient or sandbox inbox.

37. **Recommended hardening before production**
    - Add explicit error handling branches.
    - Add deduplication by website or business name.
    - Add stronger email validation regex.
    - Add rate limiting between scrape and email steps.
    - Add OpenAI prompts that strictly request valid JSON.

### Sub-workflow setup
This workflow does **not** use Execute Workflow nodes and does not invoke sub-workflows. All logic is contained in a single workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| End-to-End Local Business Lead Generation, Enrichment and Outreach Pipeline with Decodo | Workflow branding/setup context |
| Finds local business leads on Google Maps, enriches them with website and AI analysis, and sends personalized cold emails automatically | Functional summary |
| Intended for agencies, freelancers, B2B growth teams, and sales operators | Audience/context |
| Add credentials for Decodo, OpenAI, Google Sheets, and Gmail | Setup requirement |
| Replace the placeholder Google Sheet ID in the configuration node | Setup requirement |
| Create a `Leads` sheet with all required columns used in the workflow | Setup requirement |
| Update the niche, city, geo, max leads, score threshold, and sender fields | Configuration requirement |
| This template uses the Decodo community node, so it is intended for self-hosted n8n only | Platform requirement |
| Replace the placeholder image above with a real workflow screenshot before submission | Packaging/publication note |

## Additional implementation notes

- The two OpenAI nodes are present, but the export does not expose the actual prompt content. To reproduce behavior accurately, you must write prompts that force strict JSON output.
- The workflow mixes discovery/enrichment and outreach in one scheduled run. Because `Get Enriched Leads` is triggered from the first `Split In Batches`, outreach may begin during the same run after at least some enriched rows exist.
- Google Sheets is the system of record. Any schema mismatch between mapped fields and actual columns will cause failures.
- Email validation is intentionally minimal and should be strengthened for production use.
- If Gmail sends successfully but `Mark as Contacted` fails, the same lead may be emailed again in later runs unless additional safeguards are added.