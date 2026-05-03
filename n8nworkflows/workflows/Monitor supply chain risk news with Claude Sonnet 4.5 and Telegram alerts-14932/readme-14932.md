Monitor supply chain risk news with Claude Sonnet 4.5 and Telegram alerts

https://n8nworkflows.xyz/workflows/monitor-supply-chain-risk-news-with-claude-sonnet-4-5-and-telegram-alerts-14932


# Monitor supply chain risk news with Claude Sonnet 4.5 and Telegram alerts

The user wants me to analyze a complete n8n workflow JSON and produce a comprehensive, structured reference document. Let me carefully parse through the workflow and create the required sections.

Let me analyze the workflow:

**Workflow Name:** Monitor supply chain risk events from RSS feeds with Claude and Telegram

**Nodes:**
1. README (sticky note)
2. Lane: Maritime (sticky note)
3. Lane: Carrier (sticky note)
4. Lane: Geopolitical (sticky note)
5. Lane: Weather (sticky note)
6. Every 6 Hours (schedule trigger)
7. Configuration (set node)
8. RSS: gCaptain (rssFeedRead)
9. RSS: Maritime Executive (rssFeedRead)
10. RSS: Safety4Sea (rssFeedRead)
11. RSS: Splash247 (rssFeedRead)
12. RSS: The Loadstar (rssFeedRead)
13. RSS: FreightWaves (rssFeedRead)
14. RSS: Container News (rssFeedRead)
15. Google News: Tariffs & Sanctions (rssFeedRead)
16. Google News: OFAC & Export Bans (rssFeedRead)
17. RSS: GDACS (Global Disaster Alerts) (rssFeedRead)
18. RSS: NOAA Hurricane Center (rssFeedRead)
19. RSS: USGS Significant Earthquakes (rssFeedRead)
20. Merge All Feeds (merge)
21. Dedupe + Tag Source (code)
22. Sonar Analyst (Claude) (agent)
23. Claude Sonnet (Analyst) (lmChatAnthropic)
24. Parse Analyst JSON (code)
25. Severity ≥ Threshold AND Dimension Watched? (if)
26. Sonar Reporter (Claude) (agent)
27. Claude Sonnet (Reporter) (lmChatAnthropic)
28. Send Telegram Alert (telegram)
29. End (Below Threshold) (noOp)

**Connections:**
- Every 6 Hours → Configuration
- Configuration → all 12 RSS feed nodes
- All 12 RSS feed nodes → Merge All Feeds
- Merge All Feeds → Dedupe + Tag Source
- Dedupe + Tag Source → Sonar Analyst (Claude)
- Claude Sonnet (Analyst) → Sonar Analyst (Claude) [ai_languageModel connection]
- Sonar Analyst (Claude) → Parse Analyst JSON
- Parse Analyst JSON → Severity ≥ Threshold AND Dimension Watched?
- Severity ≥ Threshold AND Dimension Watched? → (true) Sonar Reporter (Claude)
- Severity ≥ Threshold AND Dimension Watched? → (false) End (Below Threshold)
- Claude Sonnet (Reporter) → Sonar Reporter (Claude) [ai_languageModel connection]
- Sonar Reporter (Claude) → Send Telegram Alert

**Sticky notes and their coverage:**
- README: covers the overall workflow area, positioned at [-592, 64] with size 848x940
- Lane: Maritime: positioned at [336, 64] with size 528x660 - covers RSS: gCaptain, RSS: Maritime Executive, RSS: Safety4Sea, RSS: Splash247
- Lane: Carrier: positioned at [336, 752] with size 528x500 - covers RSS: The Loadstar, RSS: FreightWaves, RSS: Container News
- Lane: Geopolitical: positioned at [336, 1264] with size 528x360 - covers Google News: Tariffs & Sanctions, Google News: OFAC & Export Bans
- Lane: Weather: positioned at [336, 1648] with size 528x500 - covers RSS: GDACS, RSS: NOAA Hurricane Center, RSS: USGS Significant Earthquakes

Let me now figure out which nodes each sticky note covers based on position:

- README: position [-592, 64], size 848x940 → covers x from -592 to 256, y from 64 to 1004. This likely covers "Every 6 Hours" at [240, 992] and "Configuration" at [464, 992]... Actually, Configuration at x=464 is outside 256. Let me reconsider - it may just be an overview note that doesn't specifically "cover" nodes in a technical sense. I'll attribute it generally.

- Lane: Maritime: position [336, 64], size 528x660 → covers x from 336 to 864, y from 64 to 724. RSS nodes in maritime are at x=704, y=144, 304, 464, 624. All these fall within this range. So maritime sticky covers: RSS: gCaptain, RSS: Maritime Executive, RSS: Safety4Sea, RSS: Splash247.

- Lane: Carrier: position [336, 752], size 528x500 → covers x from 336 to 864, y from 752 to 1252. RSS nodes: The Loadstar [704,832], FreightWaves [704,992], Container News [704,1152]. All within range.

- Lane: Geopolitical: position [336, 1264], size 528x360 → covers x from 336 to 864, y from 1264 to 1624. Google News: Tariffs & Sanctions [704,1344], Google News: OFAC & Export Bans [704,1504]. Both within range.

- Lane: Weather: position [336, 1648], size 528x500 → covers x from 336 to 864, y from 1648 to 2148. RSS: GDACS [704,1728], RSS: NOAA Hurricane Center [704,1888], RSS: USGS Significant Earthquakes [704,2048]. All within range.

Now let me structure the logical blocks:

1. **Trigger & Configuration** - Schedule trigger + Configuration node
2. **RSS Feed Collection (Maritime)** - 4 maritime RSS feeds
3. **RSS Feed Collection (Carrier)** - 3 carrier RSS feeds
4. **RSS Feed Collection (Geopolitical)** - 2 Google News RSS feeds
5. **RSS Feed Collection (Weather)** - 3 weather/disaster RSS feeds
6. **Feed Aggregation & Deduplication** - Merge + Dedupe nodes
7. **AI Classification (Analyst)** - Sonar Analyst agent + Claude LLM
8. **JSON Parsing & Filtering** - Parse JSON + Severity threshold IF
9. **AI Formatting (Reporter)** - Sonar Reporter agent + Claude LLM
10. **Alert Delivery** - Telegram node
11. **Silent Disposal** - NoOp end node

Let me create the full document now.

Actually, let me reconsider the block grouping. The RSS feeds across all 4 dimensions can be grouped together since they all feed into the same merge node. But the sticky notes suggest they're visually separated into lanes. I'll create blocks that reflect both the functional grouping and the visual lane separation.

Let me write the comprehensive document now.## 1. Workflow Overview

**Purpose:**  
This workflow provides an automated early-warning system for supply chain risk events. It polls 12 free public RSS feeds across four risk dimensions (Maritime, Carrier, Geopolitical, Weather), deduplicates newly seen items, classifies each with Claude Sonnet 4.5 into a structured risk assessment (dimension, severity, region, trade lanes, recommended action), and delivers HTML-formatted Telegram alerts for items that meet a configurable severity and dimension threshold. Items below threshold are silently discarded.

**Target Use Cases:**  
- Supply chain operators needing near-real-time awareness of port closures, chokepoint attacks, carrier schedule changes, sanctions, or weather disasters  
- Freight forwarders and importers/exporters monitoring specific trade lanes  
- Risk teams requiring structured, prioritized alerts rather than raw news feeds

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Trigger & Configuration | Schedules execution every 6 hours and injects tunable parameters (severity threshold, watched dimensions, watched regions, language, digest mode) |
| 2 | Maritime RSS Collection | Fetches 4 maritime-focused feeds (gCaptain, Maritime Executive, Safety4Sea, Splash247) |
| 3 | Carrier RSS Collection | Fetches 3 carrier/container feeds (The Loadstar, FreightWaves, Container News) |
| 4 | Geopolitical RSS Collection | Fetches 2 Google News searches (tariffs/sanctions, OFAC/export bans) |
| 5 | Weather RSS Collection | Fetches 3 disaster/weather feeds (GDACS, NOAA NHC, USGS Earthquakes) |
| 6 | Feed Aggregation & Deduplication | Merges all 12 streams, removes previously seen items via workflow static data, caps output at 40 items per run, tags each item with a source-inferred dimension hint |
| 7 | AI Risk Classification (Analyst) | Sends each deduplicated item to a Claude Sonnet 4.5 agent that returns structured JSON: crisis ID, dimension, severity 1–5, affected region, trade lanes, recommended action, confidence |
| 8 | JSON Parsing & Severity Filtering | Safely parses the LLM's JSON output (stripping markdown fences if present), then filters on severity ≥ threshold AND dimension is in the watched list |
| 9 | AI Alert Formatting (Reporter) | For items passing the filter, a second Claude Sonnet 4.5 agent rewrites the classification into a concise HTML Telegram message with dimension-specific emoji and severity color codes |
| 10 | Alert Delivery | Sends the HTML-formatted alert to a Telegram chat via the configured bot |
| 11 | Silent Disposal | Items below threshold terminate at a NoOp node (no action taken) |

---

## 2. Block-by-Block Analysis

---

### Block 1 – Trigger & Configuration

**Overview:**  
A schedule trigger fires the workflow every 6 hours. A Set node immediately injects configurable runtime parameters that downstream nodes reference to control alert sensitivity, scope, and language.

**Nodes Involved:**  
- Every 6 Hours  
- Configuration  

**Node Details:**

| Node | Type | Role | Configuration | Key Expressions | Input | Output | Edge Cases |
|------|------|------|---------------|-----------------|-------|--------|------------|
| Every 6 Hours | Schedule Trigger | Time-based entry point | Interval: every 6 hours | — | — | Configuration | If n8n server is down at the scheduled time, that execution is skipped (no catch-up) |
| Configuration | Set (v3.4) | Inject runtime parameters | Assignments: `severity_threshold` (number, default 3), `watched_dimensions` (string, default "maritime,carrier,geopolitical,weather"), `watched_regions` (string, default list of 13 key chokepoints/routes), `alert_language` (string, default "english"), `post_all_clear_digest` (boolean, default false) | Downstream nodes reference `$node['Configuration'].json.severity_threshold`, `$node['Configuration'].json.watched_dimensions`, `$node['Configuration'].json.watched_regions` | Every 6 Hours | All 12 RSS feed nodes | If `severity_threshold` is set outside 1–5, the IF node may never pass items or may pass everything; no validation is built in. `watched_dimensions` must use exact comma-separated dimension names (maritime, carrier, geopolitical, weather) or the contains-check in the IF node will fail. |

**Sticky Note:** README sticky note covers the overall workflow purpose and setup instructions.

---

### Block 2 – Maritime RSS Collection

**Overview:**  
Fetches the latest articles from four maritime-industry RSS feeds. These cover chokepoint disruptions, vessel incidents, port closures, and piracy.

**Nodes Involved:**  
- RSS: gCaptain  
- RSS: Maritime Executive  
- RSS: Safety4Sea  
- RSS: Splash247  

**Node Details:**

| Node | Type | Role | Feed URL | Input | Output | Edge Cases |
|------|------|------|----------|-------|--------|------------|
| RSS: gCaptain | RSS Feed Read (v1.2) | Maritime feed 1 | `https://gcaptain.com/feed/` | Configuration | Merge All Feeds | Feed may return empty items if the site is temporarily down; some items may lack `contentSnippet`, falling back to `content` in the Dedupe node |
| RSS: Maritime Executive | RSS Feed Read (v1.2) | Maritime feed 2 | `https://www.maritime-executive.com/articles.rss` | Configuration | Merge All Feeds | RSS URL uses `/articles.rss` — if the site restructures, this endpoint may 404 |
| RSS: Safety4Sea | RSS Feed Read (v1.2) | Maritime feed 3 | `https://safety4sea.com/feed/` | Configuration | Merge All Feeds | — |
| RSS: Splash247 | RSS Feed Read (v1.2) | Maritime feed 4 | `https://splash247.com/feed/` | Configuration | Merge All Feeds | — |

**Sticky Note:** "🛰️ Maritime Disruption — 4 sources covering chokepoints, port closures, vessel incidents"

---

### Block 3 – Carrier RSS Collection

**Overview:**  
Fetches three carrier and container shipping feeds covering blank sailings, equipment shortages, schedule reliability, and freight rate movements.

**Nodes Involved:**  
- RSS: The Loadstar  
- RSS: FreightWaves  
- RSS: Container News  

**Node Details:**

| Node | Type | Role | Feed URL | Input | Output | Edge Cases |
|------|------|------|----------|-------|--------|------------|
| RSS: The Loadstar | RSS Feed Read (v1.2) | Carrier feed 1 | `https://theloadstar.com/feed/` | Configuration | Merge All Feeds | Some Loadstar content may be paywalled; RSS may include partial snippets only |
| RSS: FreightWaves | RSS Feed Read (v1.2) | Carrier feed 2 | `https://www.freightwaves.com/feed/` | Configuration | Merge All Feeds | — |
| RSS: Container News | RSS Feed Read (v1.2) | Carrier feed 3 | `https://container-news.com/feed/` | Configuration | Merge All Feeds | — |

**Sticky Note:** "🚢 Carrier & Container — 3 sources covering blank sailings, equipment shortages, schedule reliability"

---

### Block 4 – Geopolitical RSS Collection

**Overview:**  
Uses two Google News RSS search queries to capture sanctions, tariffs, OFAC actions, and export bans relevant to supply chains.

**Nodes Involved:**  
- Google News: Tariffs & Sanctions  
- Google News: OFAC & Export Bans  

**Node Details:**

| Node | Type | Role | Feed URL | Input | Output | Edge Cases |
|------|------|------|----------|-------|--------|------------|
| Google News: Tariffs & Sanctions | RSS Feed Read (v1.2) | Geopolitical feed 1 | `https://news.google.com/rss/search?q=supply+chain+tariff+OR+sanctions&hl=en-US&gl=US&ceid=US:en` | Configuration | Merge All Feeds | Google News RSS can return items with limited content snippets; query syntax must use `+` for spaces and `+OR+` for OR logic |
| Google News: OFAC & Export Bans | RSS Feed Read (v1.2) | Geopolitical feed 2 | `https://news.google.com/rss/search?q=OFAC+OR+%22export+ban%22+shipping&hl=en-US&gl=US&ceid=US:en` | Configuration | Merge All Feeds | The `%22` encodes double quotes around "export ban" in the URL; modifying this query requires URL-encoding awareness |

**Sticky Note:** "⚖️ Geopolitical & Trade — 2 Google News searches for sanctions, tariffs, trade bans"

---

### Block 5 – Weather RSS Collection

**Overview:**  
Fetches three official disaster and weather alert feeds covering hurricanes, earthquakes, and multi-hazard global alerts.

**Nodes Involved:**  
- RSS: GDACS (Global Disaster Alerts)  
- RSS: NOAA Hurricane Center  
- RSS: USGS Significant Earthquakes  

**Node Details:**

| Node | Type | Role | Feed URL | Input | Output | Edge Cases |
|------|------|------|----------|-------|--------|------------|
| RSS: GDACS (Global Disaster Alerts) | RSS Feed Read (v1.2) | Weather feed 1 | `https://www.gdacs.org/xml/rss.xml` | Configuration | Merge All Feeds | GDACS XML feed may include non-maritime disasters (floods, droughts on land) — the Analyst agent will classify these |
| RSS: NOAA Hurricane Center | RSS Feed Read (v1.2) | Weather feed 2 | `https://www.nhc.noaa.gov/index-at.xml` | Configuration | Merge All Feeds | Atlantic-only; no Pacific typhoon coverage from this feed |
| RSS: USGS Significant Earthquakes | RSS Feed Read (v1.2) | Weather feed 3 | `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/significant_week.atom` | Configuration | Merge All Feeds | Atom feed format (not RSS); n8n's RSS Feed Read node handles both formats transparently. "Significant" threshold means only M4.5+ or felt events |

**Sticky Note:** "🌪️ Weather & Natural Disaster — 3 sources covering hurricanes, earthquakes, multi-hazard alerts"

---

### Block 6 – Feed Aggregation & Deduplication

**Overview:**  
Combines all 12 feed streams into one list, then removes items that have already been seen in previous runs. Each item is tagged with a dimension hint inferred from its source URL. The deduplication cache is stored in n8n workflow static data (no external database), capped at 800 URLs, and a hard limit of 40 items per run protects downstream LLM costs.

**Nodes Involved:**  
- Merge All Feeds  
- Dedupe + Tag Source  

**Node Details:**

| Node | Type | Role | Configuration | Key Logic | Input | Output | Edge Cases |
|------|------|------|---------------|-----------|-------|--------|------------|
| Merge All Feeds | Merge (v3) | Combine all feed items | Mode: Combine → Combine All (appends all inputs without matching keys) | — | All 12 RSS feed nodes | Dedupe + Tag Source | If any feed node errors, its items are simply absent — no partial-failure handling. The node expects items on input index 0 from each feed |
| Dedupe + Tag Source | Code (v2) | Deduplicate and tag dimension | Custom JavaScript (see below) | Dimension mapping: URL fragments → maritime/carrier/geopolitical/weather. Cache: `staticData.seenUrls` array. Caps: `MAX_CACHE = 800`, `MAX_ITEMS_PER_RUN = 40`. Extracts title, summary (truncated to 800 chars), URL (from `link`/`url`/`guid`), published date, source, and `source_dimension_hint` | Merge All Feeds | Sonar Analyst (Claude) | If `link`, `url`, and `guid` are all missing, the item is skipped. If static data is cleared (workflow reset), all items are treated as fresh — potential LLM cost spike. The 800-entry cap means very old URLs are eventually evicted, so a repeat item after many runs could re-appear. `MAX_ITEMS_PER_RUN = 40` is a hard break — remaining items are silently dropped |

**Dedupe Code – Key Logic Summary:**  
1. Reads `$getWorkflowStaticData('global').seenUrls` (initializes empty array on first run).  
2. Iterates all input items; for each, extracts the unique URL from `item.json.link || item.json.url || item.json.guid`.  
3. Skips if URL already in `seenUrls`.  
4. Tags the item with `source_dimension_hint` by matching the URL against a hardcoded map (e.g., `gcaptain.com` → `maritime`, `news.google.com` → `geopolitical`).  
5. Pushes the fresh item (with truncated summary at 800 chars) and adds URL to cache.  
6. Stops after 40 items.  
7. Trims `seenUrls` to last 800 entries.

---

### Block 7 – AI Risk Classification (Analyst)

**Overview:**  
A LangChain Agent node sends each deduplicated news item to Claude Sonnet 4.5 with a detailed system prompt defining the four risk dimensions, severity scale 1–5, output JSON schema, source reliability rules, recommended action rules, dimension selection heuristics, and five few-shot examples. The agent returns structured JSON only.

**Nodes Involved:**  
- Sonar Analyst (Claude)  
- Claude Sonnet (Analyst)  

**Node Details:**

| Node | Type | Role | Configuration | Key Expressions | Input | Output | Edge Cases |
|------|------|------|---------------|-----------------|-------|--------|------------|
| Sonar Analyst (Claude) | LangChain Agent (v1.7) | Classify news item into structured risk JSON | Prompt type: Define. User prompt includes `{{ $json.title }}`, `{{ $json.summary }}`, `{{ $json.url }}`, `{{ $json.published }}`, `{{ $json.source }}`, `{{ $json.source_dimension_hint }}`, `{{ $node['Configuration'].json.watched_dimensions }}`, `{{ $node['Configuration'].json.watched_regions }}`. System message defines the 4 dimensions, severity scale, JSON schema, source reliability rules, action rules, heuristics, and 5 few-shot examples | References Configuration node for thresholds and watched lists | Dedupe + Tag Source | Parse Analyst JSON | If the Anthropic API key is invalid or rate-limited, the node errors. If the LLM returns invalid JSON or wraps it in markdown fences, the downstream Parse node handles it. Timeout on the API call could cause partial run failure |
| Claude Sonnet (Analyst) | LM Chat Anthropic (v1.3) | LLM backend for the Analyst agent | Model: `claude-sonnet-4-5-20250929`. Temperature: 0.1 (near-deterministic). Max tokens: 1024 | — | Connected to Sonar Analyst via `ai_languageModel` input | Sonar Analyst consumes the LLM response | Requires Anthropic API credential configured in n8n. The low temperature (0.1) ensures consistent classification but may reduce adaptability to unusual edge-case descriptions |

**Analyst System Prompt – Key Rules Summary:**  
- **Dimensions:** maritime, carrier, geopolitical, weather (or `other`).  
- **Severity:** 1 (monitoring) through 5 (emergency).  
- **Source Reliability:** verified (official sources), probable (trade press), unconfirmed (Google News without primary source, social media). Unconfirmed sources cap severity at 2, confidence below 0.4, action = monitor.  
- **Action:** severity 1–2 → monitor; severity 3 or `carrier_advisory_issued` → alert; severity 4–5 → escalate.  
- **Heuristics:** Source URL domain strongly hints at dimension; override when article content clearly belongs elsewhere. Multi-dimension events → use upstream cause.  
- **Output Schema:** JSON with fields `crisis_id`, `primary_dimension`, `crisis_type`, `severity`, `affected_region`, `geo_node`, `trade_lanes_affected`, `source_url`, `source_reliability`, `carrier_advisory_issued`, `recommended_action`, `one_line_summary`, `confidence`.

---

### Block 8 – JSON Parsing & Severity Filtering

**Overview:**  
Safely extracts and parses the JSON from the Analyst agent's output (handling potential markdown fences), then checks two combined conditions: severity must be ≥ the configured threshold, and the item's primary dimension must appear in the watched dimensions list. Items passing both go to the Reporter; items failing either are discarded.

**Nodes Involved:**  
- Parse Analyst JSON  
- Severity ≥ Threshold AND Dimension Watched?  

**Node Details:**

| Node | Type | Role | Configuration | Key Expressions | Input | Output (true) | Output (false) | Edge Cases |
|------|------|------|---------------|-----------------|-------|---------------|----------------|------------|
| Parse Analyst JSON | Code (v2) | Extract and parse LLM JSON | Custom JS: reads `item.json.output \|\| item.json.text \|\| item.json.response \|\| JSON.stringify(item.json)`, strips ````json` and ```` ``` ```` fences, parses JSON. On parse failure, returns `{ severity: 0, primary_dimension: 'other', parse_error: e.message, raw }` | — | Sonar Analyst (Claude) | Severity ≥ Threshold AND Dimension Watched? | — | If Claude returns malformed JSON despite instructions, the fallback object with `severity: 0` will fail the threshold check and be silently discarded. The `raw` field is preserved for debugging. |
| Severity ≥ Threshold AND Dimension Watched? | IF (v2.2) | Filter items by severity and dimension | Combinator: AND. Condition 1: `$json.severity` (number) ≥ `parseInt($node['Configuration'].json.severity_threshold)`. Condition 2: `$node['Configuration'].json.watched_dimensions` (string) contains `$json.primary_dimension` | Both reference Configuration node | Parse Analyst JSON | Sonar Reporter (Claude) | End (Below Threshold) | Type validation is strict. If `severity` is not a number (e.g., string "3"), the comparison may fail. The `contains` check on a comma-separated string means partial matches are possible (e.g., dimension "car" would match "carrier") — this is a known limitation of the simple string-contains approach. |

---

### Block 9 – AI Alert Formatting (Reporter)

**Overview:**  
A second LangChain Agent reformats the classified JSON into a concise, scannable HTML message suitable for Telegram's `parse_mode=HTML`. The message includes dimension-specific emoji, severity color emoji, region, trade lanes, reliability, action, confidence, and a source link.

**Nodes Involved:**  
- Sonar Reporter (Claude)  
- Claude Sonnet (Reporter)  

**Node Details:**

| Node | Type | Role | Configuration | Key Expressions | Input | Output | Edge Cases |
|------|------|------|---------------|-----------------|-------|--------|------------|
| Sonar Reporter (Claude) | LangChain Agent (v1.7) | Format classified event into HTML Telegram alert | Prompt type: Define. User prompt includes `{{ JSON.stringify($json) }}` (the full classified JSON). System message defines the HTML template, emoji mappings (severity 1→🔵, 2→🟢, 3→🟡, 4→🟠, 5→🔴; dimension → emoji+label), rules for HTML escaping, 1000-char limit, null/empty field handling (replace with em dash `—`), and URL preservation | References the classified JSON item | Severity ≥ Threshold (true branch) | Send Telegram Alert | If the LLM wraps output in markdown fences or returns JSON instead of plain text, the Telegram node will render raw backticks. The system prompt explicitly forbids this, but occasional non-compliance is possible. HTML special characters (`<`, `>`, `&`) in source data must be escaped — the prompt instructs this but execution depends on LLM compliance |
| Claude Sonnet (Reporter) | LM Chat Anthropic (v1.3) | LLM backend for the Reporter agent | Model: `claude-sonnet-4-5-20250929`. Temperature: 0.2 (slightly more creative than Analyst). Max tokens: 600 | — | Connected to Sonar Reporter via `ai_languageModel` input | Sonar Reporter consumes the LLM response | Can reuse the same Anthropic API credential as the Analyst node. The 600 max-token limit keeps messages within Telegram's 4096-char limit and enforces brevity |

**Reporter System Prompt – Template Structure:**  
```
{severity_emoji} <b>SONAR · {dimension_emoji} {dimension_label}</b>
<i>{crisis_type} · severity {severity}/5</i>

{one_line_summary}

<b>Region:</b> {affected_region} ({geo_node})
<b>Lanes:</b> {trade_lanes_affected joined with ', ' or '—' if empty}
<b>Reliability:</b> {source_reliability}
<b>Action:</b> {recommended_action}
<b>Confidence:</b> {confidence}

<a href="{source_url}">Source ↗</a>

- Sonar Cockpit
```

---

### Block 10 – Alert Delivery

**Overview:**  
Sends the HTML-formatted alert text to a Telegram chat. The chat ID is read from the n8n environment variable `TELEGRAM_CHAT_ID`. Telegram's HTML parse mode is used, and web page previews are enabled.

**Nodes Involved:**  
- Send Telegram Alert  

**Node Details:**

| Node | Type | Role | Configuration | Key Expressions | Input | Output | Edge Cases |
|------|------|------|---------------|-----------------|-------|--------|------------|
| Send Telegram Alert | Telegram (v1.2) | Deliver HTML alert to operator chat | Text: `{{ $json.output \|\| $json.text }}` (extracts the Reporter agent's text output). Chat ID: `{{ $env.TELEGRAM_CHAT_ID }}`. Parse mode: HTML. Disable web page preview: false | Reads from env variable | Sonar Reporter (Claude) | — (end of pipeline) | If `TELEGRAM_CHAT_ID` is not set, the expression evaluates to empty and Telegram API returns an error. If the HTML contains unescaped `<` or `>`, Telegram may reject the message (400 Bad Request). Messages exceeding 4096 characters will fail. If the bot is not a member of the target chat, the API returns 403 Forbidden. The webhook ID field suggests this node could also receive incoming messages, but it is used here in send-only mode |

---

### Block 11 – Silent Disposal

**Overview:**  
Items that fail the severity/dimension check terminate here. No action is taken; the NoOp node simply marks the end of the branch.

**Nodes Involved:**  
- End (Below Threshold)  

**Node Details:**

| Node | Type | Role | Configuration | Input | Output | Edge Cases |
|------|------|------|---------------|-------|--------|------------|
| End (Below Threshold) | NoOp (v1) | Silent terminal for below-threshold items | None | Severity ≥ Threshold (false branch) | — | None — this node performs no operation and generates no output |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|-----------------|-------------|
| Every 6 Hours | Schedule Trigger | Time-based entry point | — | Configuration | README |
| Configuration | Set (v3.4) | Inject runtime parameters (severity threshold, watched dimensions, watched regions, language, digest mode) | Every 6 Hours | RSS: gCaptain; RSS: Maritime Executive; RSS: Safety4Sea; RSS: Splash247; RSS: The Loadstar; RSS: FreightWaves; RSS: Container News; Google News: Tariffs & Sanctions; Google News: OFAC & Export Bans; RSS: GDACS; RSS: NOAA Hurricane Center; RSS: USGS Significant Earthquakes | README |
| RSS: gCaptain | RSS Feed Read (v1.2) | Fetch maritime feed (gCaptain) | Configuration | Merge All Feeds | 🛰️ Maritime Disruption — 4 sources covering chokepoints, port closures, vessel incidents |
| RSS: Maritime Executive | RSS Feed Read (v1.2) | Fetch maritime feed (Maritime Executive) | Configuration | Merge All Feeds | 🛰️ Maritime Disruption — 4 sources covering chokepoints, port closures, vessel incidents |
| RSS: Safety4Sea | RSS Feed Read (v1.2) | Fetch maritime feed (Safety4Sea) | Configuration | Merge All Feeds | 🛰️ Maritime Disruption — 4 sources covering chokepoints, port closures, vessel incidents |
| RSS: Splash247 | RSS Feed Read (v1.2) | Fetch maritime feed (Splash247) | Configuration | Merge All Feeds | 🛰️ Maritime Disruption — 4 sources covering chokepoints, port closures, vessel incidents |
| RSS: The Loadstar | RSS Feed Read (v1.2) | Fetch carrier feed (The Loadstar) | Configuration | Merge All Feeds | 🚢 Carrier & Container — 3 sources covering blank sailings, equipment shortages, schedule reliability |
| RSS: FreightWaves | RSS Feed Read (v1.2) | Fetch carrier feed (FreightWaves) | Configuration | Merge All Feeds | 🚢 Carrier & Container — 3 sources covering blank sailings, equipment shortages, schedule reliability |
| RSS: Container News | RSS Feed Read (v1.2) | Fetch carrier feed (Container News) | Configuration | Merge All Feeds | 🚢 Carrier & Container — 3 sources covering blank sailings, equipment shortages, schedule reliability |
| Google News: Tariffs & Sanctions | RSS Feed Read (v1.2) | Fetch geopolitical feed (Google News: tariffs & sanctions) | Configuration | Merge All Feeds | ⚖️ Geopolitical & Trade — 2 Google News searches for sanctions, tariffs, trade bans |
| Google News: OFAC & Export Bans | RSS Feed Read (v1.2) | Fetch geopolitical feed (Google News: OFAC & export bans) | Configuration | Merge All Feeds | ⚖️ Geopolitical & Trade — 2 Google News searches for sanctions, tariffs, trade bans |
| RSS: GDACS (Global Disaster Alerts) | RSS Feed Read (v1.2) | Fetch weather feed (GDACS) | Configuration | Merge All Feeds | 🌪️ Weather & Natural Disaster — 3 sources covering hurricanes, earthquakes, multi-hazard alerts |
| RSS: NOAA Hurricane Center | RSS Feed Read (v1.2) | Fetch weather feed (NOAA NHC) | Configuration | Merge All Feeds | 🌪️ Weather & Natural Disaster — 3 sources covering hurricanes, earthquakes, multi-hazard alerts |
| RSS: USGS Significant Earthquakes | RSS Feed Read (v1.2) | Fetch weather feed (USGS) | Configuration | Merge All Feeds | 🌪️ Weather & Natural Disaster — 3 sources covering hurricanes, earthquakes, multi-hazard alerts |
| Merge All Feeds | Merge (v3) | Combine all 12 feed items into single stream | RSS: gCaptain; RSS: Maritime Executive; RSS: Safety4Sea; RSS: Splash247; RSS: The Loadstar; RSS: FreightWaves; RSS: Container News; Google News: Tariffs & Sanctions; Google News: OFAC & Export Bans; RSS: GDACS; RSS: NOAA Hurricane Center; RSS: USGS Significant Earthquakes | Dedupe + Tag Source | README |
| Dedupe + Tag Source | Code (v2) | Remove previously seen items, tag each with dimension hint, cap at 40 items | Merge All Feeds | Sonar Analyst (Claude) | README |
| Sonar Analyst (Claude) | LangChain Agent (v1.7) | Classify news item into structured risk JSON via Claude | Dedupe + Tag Source | Parse Analyst JSON | README |
| Claude Sonnet (Analyst) | LM Chat Anthropic (v1.3) | LLM backend for Analyst agent | — (ai_languageModel input from Sonar Analyst) | Sonar Analyst (Claude) | README; REVIEWER: configure your Anthropic API credential here. |
| Parse Analyst JSON | Code (v2) | Safely parse LLM JSON output | Sonar Analyst (Claude) | Severity ≥ Threshold AND Dimension Watched? | README |
| Severity ≥ Threshold AND Dimension Watched? | IF (v2.2) | Filter by severity and watched dimension | Parse Analyst JSON | (true) Sonar Reporter (Claude); (false) End (Below Threshold) | README |
| Sonar Reporter (Claude) | LangChain Agent (v1.7) | Format classified event into HTML Telegram alert | Severity ≥ Threshold (true) | Send Telegram Alert | README |
| Claude Sonnet (Reporter) | LM Chat Anthropic (v1.3) | LLM backend for Reporter agent | — (ai_languageModel input from Sonar Reporter) | Sonar Reporter (Claude) | README; REVIEWER: configure your Anthropic API credential here (can reuse the Analyst credential). |
| Send Telegram Alert | Telegram (v1.2) | Deliver HTML alert to operator Telegram chat | Sonar Reporter (Claude) | — | README |
| End (Below Threshold) | NoOp (v1) | Silent terminal for below-threshold items | Severity ≥ Threshold (false) | — | README |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Monitor supply chain risk events from RSS feeds with Claude and Telegram".

2. **Add a Schedule Trigger node.**  
   - Type: Schedule Trigger (v1.2)  
   - Name: "Every 6 Hours"  
   - Configuration: Rule → Interval → Field: Hours → Hours Interval: 6

3. **Add a Set node for configuration.**  
   - Type: Set (v3.4)  
   - Name: "Configuration"  
   - Assignments:  
     - `severity_threshold`: type Number, value `3`  
     - `watched_dimensions`: type String, value `maritime,carrier,geopolitical,weather`  
     - `watched_regions`: type String, value `Red Sea,Gulf of Aden,Strait of Hormuz,Strait of Malacca,Bab el-Mandeb,Suez Canal,Panama Canal,Black Sea,Taiwan Strait,US Gulf Coast,North Europe,Asia-Europe,Trans-Pacific`  
     - `alert_language`: type String, value `english`  
     - `post_all_clear_digest`: type Boolean, value `false`  
   - Connect: "Every 6 Hours" → "Configuration"

4. **Add 12 RSS Feed Read nodes.** For each:  
   - Type: RSS Feed Read (v1.2)  
   - Connect "Configuration" → each RSS node  
   - Connect each RSS node → "Merge All Feeds"  

   | Node Name | URL |
   |-----------|-----|
   | RSS: gCaptain | `https://gcaptain.com/feed/` |
   | RSS: Maritime Executive | `https://www.maritime-executive.com/articles.rss` |
   | RSS: Safety4Sea | `https://safety4sea.com/feed/` |
   | RSS: Splash247 | `https://splash247.com/feed/` |
   | RSS: The Loadstar | `https://theloadstar.com/feed/` |
   | RSS: FreightWaves | `https://www.freightwaves.com/feed/` |
   | RSS: Container News | `https://container-news.com/feed/` |
   | Google News: Tariffs & Sanctions | `https://news.google.com/rss/search?q=supply+chain+tariff+OR+sanctions&hl=en-US&gl=US&ceid=US:en` |
   | Google News: OFAC & Export Bans | `https://news.google.com/rss/search?q=OFAC+OR+%22export+ban%22+shipping&hl=en-US&gl=US&ceid=US:en` |
   | RSS: GDACS (Global Disaster Alerts) | `https://www.gdacs.org/xml/rss.xml` |
   | RSS: NOAA Hurricane Center | `https://www.nhc.noaa.gov/index-at.xml` |
   | RSS: USGS Significant Earthquakes | `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/significant_week.atom` |

5. **Add a Merge node.**  
   - Type: Merge (v3)  
   - Name: "Merge All Feeds"  
   - Mode: Combine → Combine All  
   - Connect all 12 RSS nodes → "Merge All Feeds"

6. **Add a Code node for deduplication.**  
   - Type: Code (v2)  
   - Name: "Dedupe + Tag Source"  
   - Language: JavaScript  
   - Paste the full deduplication code (see Block 6 analysis above for the complete logic). Key constants: `MAX_CACHE = 800`, `MAX_ITEMS_PER_RUN = 40`. The dimension map maps URL fragments to `maritime`, `carrier`, `geopolitical`, or `weather`. Uses `$getWorkflowStaticData('global')` for persistence.  
   - Connect: "Merge All Feeds" → "Dedupe + Tag Source"

7. **Add the Analyst LangChain Agent node.**  
   - Type: LangChain Agent (v1.7)  
   - Name: "Sonar Analyst (Claude)"  
   - Prompt Type: Define  
   - User prompt text:  
     ```
     Classify the following news item for supply chain risk across all 4 dimensions.

     Title: {{ $json.title }}
     Summary: {{ $json.summary }}
     URL: {{ $json.url }}
     Published: {{ $json.published }}
     Source: {{ $json.source }}
     Source-inferred dimension hint: {{ $json.source_dimension_hint }}

     Watched dimensions: {{ $node['Configuration'].json.watched_dimensions }}
     Watched regions: {{ $node['Configuration'].json.watched_regions }}

     Return ONLY valid JSON matching the schema in your system prompt. No prose, no markdown fences.
     ```  
   - System message: Paste the full Sonar Analyst system prompt (4 dimensions, severity scale 1–5, JSON schema with 13 fields, source reliability rules, action rules, heuristics, 5 few-shot examples, fallback rules). See Block 7 analysis for the complete text.  
   - Connect: "Dedupe + Tag Source" → "Sonar Analyst (Claude)"

8. **Add the Analyst LLM sub-node.**  
   - Type: LM Chat Anthropic (v1.3)  
   - Name: "Claude Sonnet (Analyst)"  
   - Model: `claude-sonnet-4-5-20250929`  
   - Temperature: 0.1  
   - Max Tokens: 1024  
   - Credential: Select or create an Anthropic API credential  
   - Connect: "Claude Sonnet (Analyst)" → "Sonar Analyst (Claude)" on the `ai_languageModel` input

9. **Add a Code node for JSON parsing.**  
   - Type: Code (v2)  
   - Name: "Parse Analyst JSON"  
   - Language: JavaScript  
   - Code: Read `item.json.output || item.json.text || item.json.response || JSON.stringify(item.json)`, strip markdown fences (` ```json ` and ` ``` `), parse JSON. On failure, return `{ severity: 0, primary_dimension: 'other', parse_error: e.message, raw }`.  
   - Connect: "Sonar Analyst (Claude)" → "Parse Analyst JSON"

10. **Add an IF node for severity and dimension filtering.**  
    - Type: IF (v2.2)  
    - Name: "Severity ≥ Threshold AND Dimension Watched?"  
    - Combinator: AND  
    - Condition 1: Left value `={{ $json.severity }}`, operator: Number ≥, right value `={{ parseInt($node['Configuration'].json.severity_threshold) }}`  
    - Condition 2: Left value `={{ $node['Configuration'].json.watched_dimensions }}`, operator: String contains, right value `={{ $json.primary_dimension }}`  
    - Connect: "Parse Analyst JSON" → "Severity ≥ Threshold AND Dimension Watched?"

11. **Add the Reporter LangChain Agent node.**  
    - Type: LangChain Agent (v1.7)  
    - Name: "Sonar Reporter (Claude)"  
    - Prompt Type: Define  
    - User prompt text:  
      ```
      Format a Telegram alert for the following classified crisis event.

      Crisis JSON:
      {{ JSON.stringify($json) }}

      Return ONLY the message text (HTML-formatted, ready for Telegram parse_mode=HTML). Do not return JSON. Do not return markdown code fences.
      ```  
    - System message: Paste the full Sonar Reporter system prompt (HTML template, severity emoji mapping 1→🔵 through 5→🔴, dimension emoji+label mapping, rules for HTML escaping, 1000-char limit, null→em dash, URL preservation). See Block 9 analysis for the complete text.  
    - Connect: "Severity ≥ Threshold AND Dimension Watched?" (true output) → "Sonar Reporter (Claude)"

12. **Add the Reporter LLM sub-node.**  
    - Type: LM Chat Anthropic (v1.3)  
    - Name: "Claude Sonnet (Reporter)"  
    - Model: `claude-sonnet-4-5-20250929`  
    - Temperature: 0.2  
    - Max Tokens: 600  
    - Credential: Select the same Anthropic API credential (or a separate one)  
    - Connect: "Claude Sonnet (Reporter)" → "Sonar Reporter (Claude)" on the `ai_languageModel` input

13. **Add a Telegram node.**  
    - Type: Telegram (v1.2)  
    - Name: "Send Telegram Alert"  
    - Text: `={{ $json.output || $json.text }}`  
    - Chat ID: `={{ $env.TELEGRAM_CHAT_ID }}`  
    - Additional Fields: Parse Mode → HTML; Disable Web Page Preview → false  
    - Credential: Select or create a Telegram Bot API credential (token from BotFather)  
    - Set the environment variable `TELEGRAM_CHAT_ID` in your n8n instance to your operator chat ID  
    - Connect: "Sonar Reporter (Claude)" → "Send Telegram Alert"

14. **Add a NoOp node for below-threshold termination.**  
    - Type: NoOp (v1)  
    - Name: "End (Below Threshold)"  
    - Connect: "Severity ≥ Threshold AND Dimension Watched?" (false output) → "End (Below Threshold)"

15. **Add sticky notes** (optional, for documentation):  
    - README note: paste the full README content (purpose, how it works, setup, requirements, customization)  
    - Maritime lane note: "🛰️ Maritime Disruption — 4 sources covering chokepoints, port closures, vessel incidents"  
    - Carrier lane note: "🚢 Carrier & Container — 3 sources covering blank sailings, equipment shortages, schedule reliability"  
    - Geopolitical lane note: "⚖️ Geopolitical & Trade — 2 Google News searches for sanctions, tariffs, trade bans"  
    - Weather lane note: "🌪️ Weather & Natural Disaster — 3 sources covering hurricanes, earthquakes, multi-hazard alerts"  

16. **Activate the workflow** by clicking "Active" toggle after a successful manual test run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| No external database is required — deduplication uses n8n workflow static data, persisted across executions | Workflow design choice |
| LLM cost is controlled by capping deduplicated items at 40 per run and the seen-URL cache at 800 entries | Dedupe Code node constants |
| The `contains` check for watched dimensions is a simple string-contains on a comma-separated list; partial substring matches are possible (e.g., a dimension "car" would match "carrier"). Consider using a more precise check if dimensions with overlapping names are added | IF node limitation |
| Google News RSS search URLs use `+` for spaces and `+OR+` for OR logic; `%22` encodes double quotes | URL encoding for Google News RSS |
| The USGS feed uses Atom format (`.atom`), which n8n's RSS Feed Read node handles transparently alongside RSS format | Feed format note |
| NOAA NHC feed covers Atlantic hurricanes only — add a Pacific typhoon feed (e.g., JTWC) if Pacific lane monitoring is needed | Coverage gap |
| To switch from Telegram to Slack, Discord, or email: replace the "Send Telegram Alert" node with the corresponding n8n node and adjust the message format accordingly (Slack uses mrkdwn, Discord uses markdown, email uses HTML) | Customization guidance |
| The `post_all_clear_digest` configuration parameter is defined but not currently consumed by any node in the workflow. It is a placeholder for a future enhancement that would send a summary digest even when no critical items are found | Unimplemented feature |
| Anthropic model identifier `claude-sonnet-4-5-20250929` must match a model available under your Anthropic API plan; if using a different Claude version, update both the Analyst and Reporter LLM sub-nodes | Model version note |
| The workflow was created by Aleksandrs Sidorecs | Project credit |