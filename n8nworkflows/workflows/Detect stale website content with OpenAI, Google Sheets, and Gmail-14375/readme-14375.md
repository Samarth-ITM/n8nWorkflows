Detect stale website content with OpenAI, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/detect-stale-website-content-with-openai--google-sheets--and-gmail-14375


# Detect stale website content with OpenAI, Google Sheets, and Gmail

# 1. Workflow Overview

This workflow scans a website’s sitemap on a weekly schedule, identifies pages that appear stale based on their `lastmod` date, fetches the content of those pages, asks OpenAI to assess whether the content is outdated, stores the findings in Google Sheets, and emails a consolidated HTML report.

Typical use cases:
- Auditing blog, documentation, or marketing pages for stale content
- Prioritizing refresh work for SEO or content operations
- Producing a recurring content-maintenance report for editors or site owners

## 1.1 Scheduled Trigger and Runtime Configuration
The workflow starts every Monday at 7:00 AM and initializes the key runtime settings:
- sitemap URL
- stale-content threshold in days
- recipient email address

## 1.2 Sitemap Retrieval and Stale Page Detection
The sitemap XML is downloaded, parsed with custom JavaScript, and filtered to retain only pages older than the configured threshold or pages missing a `lastmod` value. The resulting stale pages are sorted by age and capped at 20 items.

## 1.3 Page Fetching and AI Freshness Review
For each flagged page, the workflow downloads the page HTML, extracts the title and body, and sends a compact content preview plus metadata to an AI agent using an OpenAI chat model. The AI returns a short freshness assessment and update guidance.

## 1.4 Logging and Email Digest Generation
Each reviewed page is appended to a Google Sheet. After all rows are logged, the workflow aggregates them into a single dataset, builds a styled HTML email digest, and sends the report through Gmail.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Trigger and Configuration

### Overview
This block defines when the workflow runs and injects the main parameters used throughout the rest of the execution. It acts as the control point for customization.

### Nodes Involved
- Weekly Scan (Monday 7 AM)
- Site Configuration

### Node Details

#### 1. Weekly Scan (Monday 7 AM)
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a cron schedule.
- **Configuration choices:**  
  Configured with cron expression `0 7 * * 1`, meaning every Monday at 7:00 AM.
- **Key expressions or variables used:**  
  None in-node.
- **Input and output connections:**  
  - Input: none
  - Output: `Site Configuration`
- **Version-specific requirements:**  
  Uses node type version `1.3`.
- **Edge cases or potential failure types:**  
  - Timezone-sensitive execution: the workflow-level timezone is `America/New_York`, so the schedule runs in that timezone.
  - If the workflow is inactive, it will not trigger.
- **Sub-workflow reference:**  
  None.

#### 2. Site Configuration
- **Type and role:** `n8n-nodes-base.set`  
  Creates configuration fields used downstream.
- **Configuration choices:**  
  Assigns three static values:
  - `sitemapUrl`: `https://yoursite.com/sitemap.xml`
  - `staleDays`: `180`
  - `alertEmail`: `user@example.com`
- **Key expressions or variables used:**  
  None; values are hardcoded defaults.
- **Input and output connections:**  
  - Input: `Weekly Scan (Monday 7 AM)`
  - Output: `Fetch Sitemap XML`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Invalid sitemap URL will break later HTTP retrieval.
  - Non-numeric `staleDays` would cause logic issues in the code node.
  - Invalid email address will cause email delivery failure later.
- **Sub-workflow reference:**  
  None.

---

## Block 2 — Sitemap Retrieval and Stale Page Detection

### Overview
This block fetches the sitemap XML and parses it manually with regex-based JavaScript to identify stale pages. It also sorts and limits the candidate pages before the more expensive content analysis begins.

### Nodes Involved
- Fetch Sitemap XML
- Parse Sitemap URLs
- Any Stale Pages?

### Node Details

#### 3. Fetch Sitemap XML
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Downloads the sitemap XML content from the configured URL.
- **Configuration choices:**  
  - URL is taken from `{{$json.sitemapUrl}}`
  - Response format is explicitly set to text
  - `continueOnFail` is enabled
- **Key expressions or variables used:**  
  - `={{ $json.sitemapUrl }}`
- **Input and output connections:**  
  - Input: `Site Configuration`
  - Output: `Parse Sitemap URLs`
- **Version-specific requirements:**  
  Uses HTTP Request node version `4.2`.
- **Edge cases or potential failure types:**  
  - 404/403/500 responses
  - Timeout or DNS failure
  - Redirects to unsupported or protected endpoints
  - Because `continueOnFail` is true, downstream code may receive an error-shaped payload instead of sitemap text
- **Sub-workflow reference:**  
  None.

#### 4. Parse Sitemap URLs
- **Type and role:** `n8n-nodes-base.code`  
  Parses sitemap XML, computes staleness, and emits one item per stale page.
- **Configuration choices:**  
  The JavaScript:
  - Reads XML from `$input.first().json.data || $input.first().json.body || ''`
  - Reads `staleDays` and `alertEmail` from `Site Configuration`
  - Uses regex to extract `<url>`, `<loc>`, and `<lastmod>`
  - Marks pages stale if:
    - `daysSinceUpdate > staleDays`, or
    - `lastmod` is missing
  - Uses `daysSinceUpdate = -1` for unknown dates
  - Sorts descending by `daysSinceUpdate`
  - Returns only the top 20 stale pages
- **Key expressions or variables used:**  
  - `$('Site Configuration').first().json.staleDays`
  - `$('Site Configuration').first().json.alertEmail`
- **Input and output connections:**  
  - Input: `Fetch Sitemap XML`
  - Output: `Any Stale Pages?`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - Regex-based XML parsing is brittle for namespaces, multiline formatting, XML variations, sitemap indexes, CDATA, or escaped entities.
  - Sitemap index files are not handled recursively.
  - Invalid dates may produce `NaN` day calculations.
  - Missing response text causes zero output items.
  - Pages with missing `lastmod` are flagged as stale intentionally, but when sorted they may appear near the bottom because `-1` is used.
- **Sub-workflow reference:**  
  None.

#### 5. Any Stale Pages?
- **Type and role:** `n8n-nodes-base.if`  
  Checks whether any stale-page items were produced.
- **Configuration choices:**  
  Evaluates whether `{{$input.all().length}} > 0`.
- **Key expressions or variables used:**  
  - `={{ $input.all().length }}`
- **Input and output connections:**  
  - Input: `Parse Sitemap URLs`
  - True output: `Fetch Page Content`
  - False output: none connected
- **Version-specific requirements:**  
  Uses IF node version `2.3`.
- **Edge cases or potential failure types:**  
  - If no stale pages exist, the workflow stops silently because the false branch is not connected.
  - If upstream node emits malformed items, count logic still works but downstream semantics may not.
- **Sub-workflow reference:**  
  None.

---

## Block 3 — Page Fetching and AI Freshness Review

### Overview
This block retrieves each stale page, extracts readable content, and asks an AI model to assess freshness and recommend updates. It transforms raw web content into actionable editorial feedback.

### Nodes Involved
- Fetch Page Content
- Extract Page Text
- AI Content Freshness Analyzer
- OpenAI Chat Model
- Build Report Row

### Node Details

#### 6. Fetch Page Content
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Downloads the HTML of each stale URL.
- **Configuration choices:**  
  - URL: `{{$json.url}}`
  - Response format: text
  - Timeout: 10 seconds
  - `continueOnFail`: true
- **Key expressions or variables used:**  
  - `={{ $json.url }}`
- **Input and output connections:**  
  - Input: `Any Stale Pages?` true branch
  - Output: `Extract Page Text`
- **Version-specific requirements:**  
  Uses HTTP Request node version `4.2`.
- **Edge cases or potential failure types:**  
  - Slow pages may exceed the 10-second timeout.
  - 403 responses for bot-protected sites.
  - Redirect loops, SSL issues, anti-bot pages.
  - Because `continueOnFail` is enabled, extraction may receive error payloads rather than HTML.
- **Sub-workflow reference:**  
  None.

#### 7. Extract Page Text
- **Type and role:** `n8n-nodes-base.html`  
  Extracts the page `<title>` and `<body>` from the fetched HTML.
- **Configuration choices:**  
  Operation is `extractHtmlContent` with two CSS selectors:
  - `title` → `title`
  - `body` → `body`
  `continueOnFail` is enabled.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Fetch Page Content`
  - Output: `AI Content Freshness Analyzer`
- **Version-specific requirements:**  
  Uses HTML node version `1.2`.
- **Edge cases or potential failure types:**  
  - Invalid HTML or non-HTML responses
  - Body extraction may return very large text
  - Dynamic sites rendered client-side may yield incomplete content
  - Error payloads from the previous HTTP node may lead to empty extraction results
- **Sub-workflow reference:**  
  None.

#### 8. AI Content Freshness Analyzer
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent node that evaluates the freshness of page content and produces a concise recommendation.
- **Configuration choices:**  
  - Prompt type: defined directly in the node
  - User prompt includes:
    - days since sitemap update
    - page URL
    - extracted title
    - first 1000 characters of body text
    - requested output dimensions: outdated references, relevance, priority level, update suggestions
    - response constrained to 4–5 sentences
  - System message defines the role as a practical content strategist
- **Key expressions or variables used:**  
  - `{{ $('Parse Sitemap URLs').item.json.daysSinceUpdate }}`
  - `{{ $('Parse Sitemap URLs').item.json.url }}`
  - `{{ $json.title || 'No title found' }}`
  - `{{ ($json.body || 'Could not extract content').substring(0, 1000) }}`
- **Input and output connections:**  
  - Main input: `Extract Page Text`
  - AI language model input: `OpenAI Chat Model`
  - Main output: `Build Report Row`
- **Version-specific requirements:**  
  Uses LangChain Agent node version `3.1`. Requires compatible AI subnode connection.
- **Edge cases or potential failure types:**  
  - Missing or empty body/title reduces review quality.
  - Prompt may reference `.item` from `Parse Sitemap URLs`; item linking generally works in linear item flow, but can be fragile if batching/merging patterns change.
  - Model/API quota or auth errors propagate from the connected chat model.
  - Token limits are mitigated by truncating body content to 1000 characters.
- **Sub-workflow reference:**  
  None.

#### 9. OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM backend for the AI agent.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - No additional model options configured
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to: `AI Content Freshness Analyzer`
- **Version-specific requirements:**  
  Uses node version `1.3`. Requires valid OpenAI credentials and access to the selected model.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Rate limits or quota exhaustion
  - Model availability changes
  - Organization/project restrictions
- **Sub-workflow reference:**  
  None.

#### 10. Build Report Row
- **Type and role:** `n8n-nodes-base.set`  
  Normalizes the data into a row structure suitable for Google Sheets.
- **Configuration choices:**  
  Creates:
  - `pageUrl` from parsed sitemap item
  - `lastModified` from parsed sitemap item
  - `daysSinceUpdate` from parsed sitemap item
  - `aiReview` from AI output
- **Key expressions or variables used:**  
  - `={{ $('Parse Sitemap URLs').item.json.url }}`
  - `={{ $('Parse Sitemap URLs').item.json.lastModified }}`
  - `={{ $('Parse Sitemap URLs').item.json.daysSinceUpdate }}`
  - `={{ $json.output }}`
- **Input and output connections:**  
  - Input: `AI Content Freshness Analyzer`
  - Output: `Save to Content Audit Sheet`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - If AI node returns a different property than `output`, the review field may be blank.
  - Item-linking dependence on `Parse Sitemap URLs` can break if execution topology changes.
- **Sub-workflow reference:**  
  None.

---

## Block 4 — Logging and Email Digest Generation

### Overview
This block stores each AI-reviewed stale page in Google Sheets, then collects all rows into one structure, formats an HTML digest, and sends it by email. It is the reporting and notification layer of the workflow.

### Nodes Involved
- Save to Content Audit Sheet
- Combine Into One
- Build Email Body
- Email Content Audit Report

### Node Details

#### 11. Save to Content Audit Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Appends one row per reviewed page to a Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Target spreadsheet: specified by URL
  - Target sheet: `gid=0`, cached as `ContentAudit`
  - Explicit field mapping:
    - `page_url` ← `{{$json.pageUrl}}`
    - `ai_review` ← `{{$json.aiReview}}`
    - `scan_date` ← `{{$now.toISO()}}`
    - `last_modified` ← `{{$json.lastModified}}`
    - `days_since_update` ← `{{$json.daysSinceUpdate}}`
- **Key expressions or variables used:**  
  - `={{ $json.pageUrl }}`
  - `={{ $json.aiReview }}`
  - `={{ $now.toISO() }}`
  - `={{ $json.lastModified }}`
  - `={{ $json.daysSinceUpdate }}`
- **Input and output connections:**  
  - Input: `Build Report Row`
  - Output: `Combine Into One`
- **Version-specific requirements:**  
  Uses Google Sheets node version `4.7`. Requires OAuth2 credentials and write access to the spreadsheet.
- **Edge cases or potential failure types:**  
  - Missing spreadsheet permissions
  - Invalid spreadsheet URL or wrong tab selection
  - Schema mismatch if headers do not exist as expected
  - API quota issues
- **Sub-workflow reference:**  
  None.

#### 12. Combine Into One
- **Type and role:** `n8n-nodes-base.aggregate`  
  Aggregates all processed items into a single item containing an array of item data.
- **Configuration choices:**  
  Aggregate mode: `aggregateAllItemData`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Save to Content Audit Sheet`
  - Output: `Build Email Body`
- **Version-specific requirements:**  
  Uses Aggregate node version `1`.
- **Edge cases or potential failure types:**  
  - If no items reach this node, it will not produce an email body.
  - Large result sets can increase payload size, though this workflow limits stale pages to 20.
- **Sub-workflow reference:**  
  None.

#### 13. Build Email Body
- **Type and role:** `n8n-nodes-base.code`  
  Generates a styled HTML email summarizing all stale pages.
- **Configuration choices:**  
  The JavaScript:
  - Reads aggregated items from `$input.first().json.data || []`
  - Reads `staleDays` from `Site Configuration`
  - Builds an HTML report
  - Shows total flagged pages
  - For each page:
    - derives a path from the full URL
    - displays “Last updated” as days ago or Unknown
    - colors entries by age:
      - `> 365`: red
      - `> 270`: orange
      - otherwise yellow
  - Includes a footer mentioning the threshold and the Google Sheet
- **Key expressions or variables used:**  
  - `$('Site Configuration').first().json.staleDays`
- **Input and output connections:**  
  - Input: `Combine Into One`
  - Output: `Email Content Audit Report`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - HTML is assembled by string interpolation without escaping content; unusual AI output could affect formatting.
  - Unknown dates (`-1`) render as `Unknown`.
  - The “no stale pages” branch exists in code, but in practice this node is only reached when at least one item has been processed.
- **Sub-workflow reference:**  
  None.

#### 14. Email Content Audit Report
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends the final HTML report by email.
- **Configuration choices:**  
  - Recipient: `{{$('Site Configuration').first().json.alertEmail}}`
  - Subject: `Weekly Stale Content Report - {{ $now.format('yyyy-MM-dd') }}`
  - Message body: `{{$json.emailBody}}`
- **Key expressions or variables used:**  
  - `={{ $('Site Configuration').first().json.alertEmail }}`
  - `=Weekly Stale Content Report - {{ $now.format('yyyy-MM-dd') }}`
  - `={{ $json.emailBody }}`
- **Input and output connections:**  
  - Input: `Build Email Body`
  - Output: none
- **Version-specific requirements:**  
  Uses Gmail node version `2.2`. Requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - OAuth token expiry or insufficient Gmail scopes
  - Sending limits or anti-spam enforcement
  - Invalid recipient address
  - Depending on Gmail node behavior/settings, HTML rendering may depend on content-type handling; test a real send after setup
- **Sub-workflow reference:**  
  None.

---

## Block 5 — In-Canvas Documentation Notes

### Overview
These nodes are not part of execution logic. They document the workflow purpose, setup steps, and the functional grouping of nodes directly on the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### 15. Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`  
  General workflow documentation panel.
- **Configuration choices:**  
  Contains:
  - workflow summary
  - setup checklist
  - customization ideas
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses version `1`.
- **Edge cases or potential failure types:**  
  None; visual-only node.
- **Sub-workflow reference:**  
  None.

#### 16. Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the trigger/configuration section.
- **Configuration choices:**  
  Explains that the workflow runs every Monday at 7 AM and that key parameters are set in `Site Configuration`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 17. Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the sitemap fetch/parse section.
- **Configuration choices:**  
  Notes that the workflow extracts URLs and `lastmod`, sorts by staleness, and limits results to 20.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 18. Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the content extraction and AI analysis section.
- **Configuration choices:**  
  Notes that each stale page is fetched, parsed, and reviewed by AI.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 19. Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the logging and email section.
- **Configuration choices:**  
  Notes that the workflow logs to Google Sheets and emails a color-coded digest.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Scan (Monday 7 AM) | Schedule Trigger | Starts workflow every Monday at 7 AM |  | Site Configuration | ## Trigger and configuration<br>Runs every Monday at 7 AM. Set your sitemap URL, staleness threshold, and alert email in the Site Configuration node. |
| Site Configuration | Set | Stores sitemap URL, stale threshold, and alert email | Weekly Scan (Monday 7 AM) | Fetch Sitemap XML | ## Trigger and configuration<br>Runs every Monday at 7 AM. Set your sitemap URL, staleness threshold, and alert email in the Site Configuration node.<br>## Stale Content Detector for Websites<br>### How it works<br>1. Every Monday at 7 AM, the workflow fetches your sitemap.xml and extracts all URLs with their last-modified dates<br>2. Pages not updated within your configured threshold (default: 180 days) are flagged and sorted most-stale-first<br>3. Each stale page is fetched and its content is analyzed by an AI agent that rates freshness as LOW, MEDIUM, HIGH, or CRITICAL with specific update suggestions<br>4. Results are logged to Google Sheets and a color-coded HTML email digest is sent with all flagged pages and their AI verdicts<br>### Setup steps<br>- [ ] Open **Site Configuration** and set your sitemapUrl, staleDays (default: 180), and alertEmail<br>- [ ] Create a Google Sheet with a **ContentAudit** tab (columns: scan_date, page_url, last_modified, days_since_update, ai_review)<br>- [ ] Paste your Google Sheet URL into the **Save to Content Audit Sheet** node<br>- [ ] Connect Gmail OAuth2 credentials on the **Email Content Audit Report** node<br>- [ ] Connect Google Sheets credentials<br>- [ ] Connect OpenAI API credentials on the **OpenAI Chat Model** node<br>### Customization<br>Change the staleDays threshold in Site Configuration. Increase the page limit above 20 in the Code node for larger sites. Add URL path filters to focus on blog posts or docs only. Replace Gmail with Slack for team notifications. |
| Fetch Sitemap XML | HTTP Request | Downloads sitemap XML | Site Configuration | Parse Sitemap URLs | ## Fetch and parse sitemap<br>Fetches your sitemap.xml, extracts URLs with last-modified dates, and flags pages not updated within the configured threshold. Results are sorted most-stale-first, capped at 20. |
| Parse Sitemap URLs | Code | Extracts URLs and computes staleness | Fetch Sitemap XML | Any Stale Pages? | ## Fetch and parse sitemap<br>Fetches your sitemap.xml, extracts URLs with last-modified dates, and flags pages not updated within the configured threshold. Results are sorted most-stale-first, capped at 20. |
| Any Stale Pages? | If | Stops processing when no stale URLs are found | Parse Sitemap URLs | Fetch Page Content | ## Fetch and parse sitemap<br>Fetches your sitemap.xml, extracts URLs with last-modified dates, and flags pages not updated within the configured threshold. Results are sorted most-stale-first, capped at 20. |
| Fetch Page Content | HTTP Request | Downloads each stale webpage | Any Stale Pages? | Extract Page Text | ## Content extraction and AI review<br>Fetches each stale page, extracts title and body text, then sends it to an AI agent that rates content freshness and provides specific update suggestions. |
| Extract Page Text | HTML | Extracts title and body content from HTML | Fetch Page Content | AI Content Freshness Analyzer | ## Content extraction and AI review<br>Fetches each stale page, extracts title and body text, then sends it to an AI agent that rates content freshness and provides specific update suggestions. |
| AI Content Freshness Analyzer | LangChain Agent | Reviews stale content and suggests updates | Extract Page Text; OpenAI Chat Model | Build Report Row | ## Content extraction and AI review<br>Fetches each stale page, extracts title and body text, then sends it to an AI agent that rates content freshness and provides specific update suggestions. |
| OpenAI Chat Model | OpenAI Chat Model | Supplies LLM to the agent node |  | AI Content Freshness Analyzer | ## Content extraction and AI review<br>Fetches each stale page, extracts title and body text, then sends it to an AI agent that rates content freshness and provides specific update suggestions.<br>## Stale Content Detector for Websites<br>### How it works<br>1. Every Monday at 7 AM, the workflow fetches your sitemap.xml and extracts all URLs with their last-modified dates<br>2. Pages not updated within your configured threshold (default: 180 days) are flagged and sorted most-stale-first<br>3. Each stale page is fetched and its content is analyzed by an AI agent that rates freshness as LOW, MEDIUM, HIGH, or CRITICAL with specific update suggestions<br>4. Results are logged to Google Sheets and a color-coded HTML email digest is sent with all flagged pages and their AI verdicts<br>### Setup steps<br>- [ ] Open **Site Configuration** and set your sitemapUrl, staleDays (default: 180), and alertEmail<br>- [ ] Create a Google Sheet with a **ContentAudit** tab (columns: scan_date, page_url, last_modified, days_since_update, ai_review)<br>- [ ] Paste your Google Sheet URL into the **Save to Content Audit Sheet** node<br>- [ ] Connect Gmail OAuth2 credentials on the **Email Content Audit Report** node<br>- [ ] Connect Google Sheets credentials<br>- [ ] Connect OpenAI API credentials on the **OpenAI Chat Model** node<br>### Customization<br>Change the staleDays threshold in Site Configuration. Increase the page limit above 20 in the Code node for larger sites. Add URL path filters to focus on blog posts or docs only. Replace Gmail with Slack for team notifications. |
| Build Report Row | Set | Maps AI output and metadata into report fields | AI Content Freshness Analyzer | Save to Content Audit Sheet | ## Content extraction and AI review<br>Fetches each stale page, extracts title and body text, then sends it to an AI agent that rates content freshness and provides specific update suggestions. |
| Save to Content Audit Sheet | Google Sheets | Appends audit rows to spreadsheet | Build Report Row | Combine Into One | ## Logging and email report<br>Logs each reviewed page to Google Sheets with the full AI analysis. Results are aggregated into a color-coded HTML email digest and sent to your alert address.<br>## Stale Content Detector for Websites<br>### How it works<br>1. Every Monday at 7 AM, the workflow fetches your sitemap.xml and extracts all URLs with their last-modified dates<br>2. Pages not updated within your configured threshold (default: 180 days) are flagged and sorted most-stale-first<br>3. Each stale page is fetched and its content is analyzed by an AI agent that rates freshness as LOW, MEDIUM, HIGH, or CRITICAL with specific update suggestions<br>4. Results are logged to Google Sheets and a color-coded HTML email digest is sent with all flagged pages and their AI verdicts<br>### Setup steps<br>- [ ] Open **Site Configuration** and set your sitemapUrl, staleDays (default: 180), and alertEmail<br>- [ ] Create a Google Sheet with a **ContentAudit** tab (columns: scan_date, page_url, last_modified, days_since_update, ai_review)<br>- [ ] Paste your Google Sheet URL into the **Save to Content Audit Sheet** node<br>- [ ] Connect Gmail OAuth2 credentials on the **Email Content Audit Report** node<br>- [ ] Connect Google Sheets credentials<br>- [ ] Connect OpenAI API credentials on the **OpenAI Chat Model** node<br>### Customization<br>Change the staleDays threshold in Site Configuration. Increase the page limit above 20 in the Code node for larger sites. Add URL path filters to focus on blog posts or docs only. Replace Gmail with Slack for team notifications. |
| Combine Into One | Aggregate | Collects all report rows into a single array | Save to Content Audit Sheet | Build Email Body | ## Logging and email report<br>Logs each reviewed page to Google Sheets with the full AI analysis. Results are aggregated into a color-coded HTML email digest and sent to your alert address. |
| Build Email Body | Code | Generates HTML digest email | Combine Into One | Email Content Audit Report | ## Logging and email report<br>Logs each reviewed page to Google Sheets with the full AI analysis. Results are aggregated into a color-coded HTML email digest and sent to your alert address. |
| Email Content Audit Report | Gmail | Sends final HTML report email | Build Email Body |  | ## Logging and email report<br>Logs each reviewed page to Google Sheets with the full AI analysis. Results are aggregated into a color-coded HTML email digest and sent to your alert address.<br>## Stale Content Detector for Websites<br>### How it works<br>1. Every Monday at 7 AM, the workflow fetches your sitemap.xml and extracts all URLs with their last-modified dates<br>2. Pages not updated within your configured threshold (default: 180 days) are flagged and sorted most-stale-first<br>3. Each stale page is fetched and its content is analyzed by an AI agent that rates freshness as LOW, MEDIUM, HIGH, or CRITICAL with specific update suggestions<br>4. Results are logged to Google Sheets and a color-coded HTML email digest is sent with all flagged pages and their AI verdicts<br>### Setup steps<br>- [ ] Open **Site Configuration** and set your sitemapUrl, staleDays (default: 180), and alertEmail<br>- [ ] Create a Google Sheet with a **ContentAudit** tab (columns: scan_date, page_url, last_modified, days_since_update, ai_review)<br>- [ ] Paste your Google Sheet URL into the **Save to Content Audit Sheet** node<br>- [ ] Connect Gmail OAuth2 credentials on the **Email Content Audit Report** node<br>- [ ] Connect Google Sheets credentials<br>- [ ] Connect OpenAI API credentials on the **OpenAI Chat Model** node<br>### Customization<br>Change the staleDays threshold in Site Configuration. Increase the page limit above 20 in the Code node for larger sites. Add URL path filters to focus on blog posts or docs only. Replace Gmail with Slack for team notifications. |
| Sticky Note | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for trigger/config section |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for sitemap section |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for AI review section |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for reporting section |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Stale Content Detector for Websites`.
   - Set the workflow timezone to `America/New_York` if you want exact parity with the original behavior.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Weekly Scan (Monday 7 AM)`
   - Set the schedule rule to cron expression:
     - `0 7 * * 1`
   - This makes the workflow run every Monday at 7:00 AM.

3. **Add a Set node for configuration**
   - Node type: **Set**
   - Name: `Site Configuration`
   - Add these fields:
     - `sitemapUrl` as string, e.g. `https://yoursite.com/sitemap.xml`
     - `staleDays` as number, e.g. `180`
     - `alertEmail` as string, e.g. `user@example.com`
   - Connect:
     - `Weekly Scan (Monday 7 AM)` → `Site Configuration`

4. **Add an HTTP Request node to fetch the sitemap**
   - Node type: **HTTP Request**
   - Name: `Fetch Sitemap XML`
   - Method: `GET`
   - URL expression:
     - `{{ $json.sitemapUrl }}`
   - Response format: **Text**
   - Enable **Continue On Fail**
   - Connect:
     - `Site Configuration` → `Fetch Sitemap XML`

5. **Add a Code node to parse the sitemap**
   - Node type: **Code**
   - Name: `Parse Sitemap URLs`
   - Paste JavaScript equivalent to this logic:
     1. Read XML from the HTTP response text
     2. Read `staleDays` and `alertEmail` from `Site Configuration`
     3. Extract each `<url>` block
     4. From each block, extract `<loc>` and optional `<lastmod>`
     5. If `lastmod` exists:
        - compute `daysSinceUpdate`
        - mark stale if `daysSinceUpdate > staleDays`
     6. If `lastmod` does not exist:
        - mark the page stale
        - set `daysSinceUpdate` to `-1`
     7. Emit items with:
        - `url`
        - `lastModified`
        - `daysSinceUpdate`
        - `staleDays`
        - `alertEmail`
     8. Sort descending by `daysSinceUpdate`
     9. Return only the first 20 items
   - Use expressions inside the code like:
     - `$('Site Configuration').first().json.staleDays`
     - `$('Site Configuration').first().json.alertEmail`
   - Connect:
     - `Fetch Sitemap XML` → `Parse Sitemap URLs`

6. **Add an IF node to stop when there are no stale pages**
   - Node type: **If**
   - Name: `Any Stale Pages?`
   - Condition:
     - left value: `{{ $input.all().length }}`
     - operator: **greater than**
     - right value: `0`
   - Connect:
     - `Parse Sitemap URLs` → `Any Stale Pages?`
   - Only use the **true** output for the next step.

7. **Add an HTTP Request node to fetch each stale page**
   - Node type: **HTTP Request**
   - Name: `Fetch Page Content`
   - Method: `GET`
   - URL expression:
     - `{{ $json.url }}`
   - Response format: **Text**
   - Timeout: `10000` ms
   - Enable **Continue On Fail**
   - Connect:
     - `Any Stale Pages?` true branch → `Fetch Page Content`

8. **Add an HTML node to extract title and body**
   - Node type: **HTML**
   - Name: `Extract Page Text`
   - Operation: **Extract HTML Content**
   - Add extraction values:
     - key `title` with CSS selector `title`
     - key `body` with CSS selector `body`
   - Enable **Continue On Fail**
   - Connect:
     - `Fetch Page Content` → `Extract Page Text`

9. **Add an OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model** from the LangChain/AI nodes
   - Name: `OpenAI Chat Model`
   - Select model: `gpt-4o-mini`
   - Attach valid **OpenAI API credentials**
   - No special model options are required for parity.

10. **Add an AI Agent node**
    - Node type: **AI Agent** / LangChain Agent
    - Name: `AI Content Freshness Analyzer`
    - Prompt type: define directly in the node
    - Add this prompt structure:
      - mention days since update from `Parse Sitemap URLs`
      - include page URL
      - include title
      - include first 1000 chars of extracted body
      - ask for:
        1. outdated references
        2. relevance check
        3. refresh priority: LOW / MEDIUM / HIGH / CRITICAL
        4. specific update suggestions
      - instruct response length: 4–5 sentences maximum
    - Add a system message similar to:
      - “You are a content strategist who audits web pages for freshness and accuracy. Be practical and specific in your recommendations. Only flag things that genuinely need updating.”
    - Connect:
      - `Extract Page Text` → `AI Content Freshness Analyzer`
      - `OpenAI Chat Model` → AI language model input of `AI Content Freshness Analyzer`

11. **Use these prompt expressions in the AI node**
    - Days stale:
      - `{{ $('Parse Sitemap URLs').item.json.daysSinceUpdate }}`
    - URL:
      - `{{ $('Parse Sitemap URLs').item.json.url }}`
    - Title:
      - `{{ $json.title || 'No title found' }}`
    - Body preview:
      - `{{ ($json.body || 'Could not extract content').substring(0, 1000) }}`
    - This preserves the original item-linking design.

12. **Add a Set node to prepare report rows**
    - Node type: **Set**
    - Name: `Build Report Row`
    - Add these fields:
      - `pageUrl` = `{{ $('Parse Sitemap URLs').item.json.url }}`
      - `lastModified` = `{{ $('Parse Sitemap URLs').item.json.lastModified }}`
      - `daysSinceUpdate` = `{{ $('Parse Sitemap URLs').item.json.daysSinceUpdate }}`
      - `aiReview` = `{{ $json.output }}`
    - Connect:
      - `AI Content Freshness Analyzer` → `Build Report Row`

13. **Create the Google Sheet before configuring the next node**
    - In Google Sheets, create a spreadsheet with a tab named `ContentAudit` or use the first sheet.
    - Ensure the header row contains:
      - `scan_date`
      - `page_url`
      - `last_modified`
      - `days_since_update`
      - `ai_review`

14. **Add a Google Sheets node**
    - Node type: **Google Sheets**
    - Name: `Save to Content Audit Sheet`
    - Operation: **Append**
    - Connect valid **Google Sheets OAuth2 credentials**
    - Select your spreadsheet by URL or ID
    - Select the target tab/sheet
    - Set manual field mapping:
      - `page_url` = `{{ $json.pageUrl }}`
      - `ai_review` = `{{ $json.aiReview }}`
      - `scan_date` = `{{ $now.toISO() }}`
      - `last_modified` = `{{ $json.lastModified }}`
      - `days_since_update` = `{{ $json.daysSinceUpdate }}`
    - Connect:
      - `Build Report Row` → `Save to Content Audit Sheet`

15. **Add an Aggregate node**
    - Node type: **Aggregate**
    - Name: `Combine Into One`
    - Aggregate mode: **Aggregate All Item Data**
    - Connect:
      - `Save to Content Audit Sheet` → `Combine Into One`

16. **Add a Code node to generate the email HTML**
    - Node type: **Code**
    - Name: `Build Email Body`
    - Implement logic that:
      - reads aggregated items from `$input.first().json.data || []`
      - gets `staleDays` from `Site Configuration`
      - creates an HTML wrapper
      - prints total flagged pages
      - loops through each item
      - derives a path from the page URL
      - renders “days ago” or `Unknown`
      - assigns color by age:
        - red if over 365 days
        - orange if over 270 days
        - yellow otherwise
      - adds a footer referencing the threshold
      - returns one item with:
        - `emailBody`
        - `totalPages`
    - Connect:
      - `Combine Into One` → `Build Email Body`

17. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: `Email Content Audit Report`
    - Connect valid **Gmail OAuth2 credentials**
    - Send To:
      - `{{ $('Site Configuration').first().json.alertEmail }}`
    - Subject:
      - `Weekly Stale Content Report - {{ $now.format('yyyy-MM-dd') }}`
    - Message:
      - `{{ $json.emailBody }}`
    - Connect:
      - `Build Email Body` → `Email Content Audit Report`

18. **Optionally add sticky notes for maintainability**
    - Add notes for:
      - trigger/configuration
      - sitemap parsing
      - AI review
      - logging/email
    - Include operational setup instructions for future maintainers.

19. **Configure credentials**
    - **OpenAI**
      - API key with access to `gpt-4o-mini`
    - **Google Sheets OAuth2**
      - Must have read/write access to the chosen spreadsheet
    - **Gmail OAuth2**
      - Must have permission to send mail from the chosen account

20. **Test with manual execution**
    - Replace the placeholder sitemap URL with a real public sitemap.
    - Run the workflow manually.
    - Verify:
      - sitemap fetch succeeds
      - stale pages are emitted
      - page fetches do not fail on bot protection
      - AI node returns output
      - rows appear in Sheets
      - HTML email renders correctly in Gmail

21. **Activate the workflow**
    - After validating the run, activate the workflow so the schedule becomes live.

## Reproduction constraints and implementation notes
- The workflow assumes a standard XML sitemap with `<url>`, `<loc>`, and optional `<lastmod>`.
- It does **not** handle sitemap index recursion.
- It intentionally limits analysis to 20 stale pages for cost and speed control.
- If you need “no stale pages” notifications, add a node on the false branch of `Any Stale Pages?`.
- If item-linking becomes unreliable after modifications, consider carrying metadata forward directly in each item instead of referencing `$('Parse Sitemap URLs').item`.

## Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require any child workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Stale Content Detector for Websites — every Monday at 7 AM, the workflow fetches your sitemap.xml, extracts last-modified dates, flags pages older than the threshold, analyzes stale pages with AI, logs results to Google Sheets, and emails a color-coded digest. | General workflow purpose |
| Setup checklist: configure `sitemapUrl`, `staleDays`, and `alertEmail` in `Site Configuration`. | Operational setup |
| Create a Google Sheet with a `ContentAudit` tab and columns: `scan_date`, `page_url`, `last_modified`, `days_since_update`, `ai_review`. | Google Sheets prerequisite |
| Paste your Google Sheet URL into the `Save to Content Audit Sheet` node. | Google Sheets configuration |
| Connect Gmail OAuth2 credentials on `Email Content Audit Report`. | Email credential setup |
| Connect Google Sheets credentials. | Sheets credential setup |
| Connect OpenAI API credentials on `OpenAI Chat Model`. | AI credential setup |
| Customization options: change `staleDays`, increase the 20-page limit in the parsing code, add URL path filters for blog/docs, or replace Gmail with Slack. | Extension ideas |