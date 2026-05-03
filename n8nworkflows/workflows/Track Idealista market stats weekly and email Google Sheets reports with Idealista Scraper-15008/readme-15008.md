Track Idealista market stats weekly and email Google Sheets reports with Idealista Scraper

https://n8nworkflows.xyz/workflows/track-idealista-market-stats-weekly-and-email-google-sheets-reports-with-idealista-scraper-15008


# Track Idealista market stats weekly and email Google Sheets reports with Idealista Scraper

### 1. Workflow Overview

This workflow performs **weekly automated real estate market analysis** by scraping property listings from Idealista (a major Southern European property portal with no official API), computing statistical summaries per market, generating a styled HTML comparison report, delivering it via email, and persisting the structured data to Google Sheets for longitudinal trend tracking.

**Target use cases:**
- Real estate investors monitoring price trends across Spanish cities
- Market analysts comparing Madrid vs Barcelona sale prices over time
- Rental yield analysts (by switching the operation from `sale` to `rent`)
- Extension to additional markets (Valencia, Rome, Lisbon, Milan)

**Logical blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Scheduling & Trigger | Fires the workflow every Monday at 08:00 |
| 2 | Market Scraping | Fetches property listings from two Idealista markets in parallel |
| 3 | Statistical Analysis | Computes per-market statistics (avg/median price, price per m², size, rooms, inventory) |
| 4 | Data Merging | Combines both market analyses into a single dataset |
| 5 | Report Generation | Builds a professionally formatted HTML comparison table |
| 6 | Delivery & Logging | Emails the HTML report and appends structured stats to Google Sheets |

**Data flow diagram:**

```
[Schedule Trigger]
       │
       ├──► [Scrape Madrid]    ──► [Analyze Madrid]    ──┐
       │                                                  ├──► [Merge Market Data] ──┬──► [Build HTML Report] ──► [Email Report]
       └──► [Scrape Barcelona]──► [Analyze Barcelona]──┘                           │
                                                                                   └──► [Log to Market History]
```

**Key dependency:** This workflow requires the **`n8n-nodes-idealista-scraper`** community node, which relies on an Apify API token. It will **not** run on n8n Cloud (community nodes are unsupported there) — a self-hosted n8n instance is mandatory.

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Scheduling & Trigger

**Overview:** A cron-based schedule trigger that starts the entire pipeline every Monday at 08:00 (server timezone). No manual intervention is needed once the workflow is activated.

**Nodes Involved:** Every Monday 8am

| Attribute | Detail |
|-----------|--------|
| **Node** | Every Monday 8am |
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.2) |
| **Technical Role** | Workflow entry point; emits a single empty item to start execution |
| **Configuration** | Cron expression: `0 8 * * 1` — every Monday at minute 0, hour 8 |
| **Key Expressions** | None |
| **Input** | None (trigger node) |
| **Output** | Two parallel connections: → Scrape Madrid, → Scrape Barcelona |
| **Version Requirements** | v1.2 of the schedule trigger supports cron expressions natively |
| **Edge Cases / Failures** | If the n8n instance is down at 08:00, the trigger is missed (no retry). Timezone depends on the n8n server's TZ environment variable — verify it matches the intended timezone. No authentication required. |

---

#### Block 2 — Market Scraping

**Overview:** Two parallel Idealista Scraper nodes fetch property-for-sale listings from Madrid and Barcelona. Each scrapes 3 pages (~120 properties per city). The scrapers use API-based extraction via the Apify platform, not browser automation, so they are resilient to site layout changes.

**Nodes Involved:** Scrape Madrid, Scrape Barcelona

| Attribute | Detail |
|-----------|--------|
| **Node** | Scrape Madrid |
| **Type** | `n8n-nodes-idealista-scraper.idealistaScraper` (v1) |
| **Technical Role** | Scrapes Idealista sale listings for the Madrid market |
| **Configuration** | Operation: `sale`; `numPages`: 3; All filter groups (size, price, rental, feature, advanced, condition, floorTime, propertyType, bedroomBathroom) left at default (empty = no filters); No explicit `locationId` or `locationName` set (defaults to Madrid via the node's internal default) |
| **Key Expressions** | None |
| **Input** | From: Every Monday 8am |
| **Output** | → Analyze Madrid |
| **Credential** | Requires an **Apify API** credential (API token from [Apify Console](https://console.apify.com/account/integrations)) |
| **Edge Cases / Failures** | Apify API token missing/invalid → authentication error. Apify account out of credits → HTTP 402/429. Location ID not set may default to Madrid but could fail if the node has no default — verify on first run. Network timeout if Apify platform is slow. If fewer than 3 pages of results exist, the node returns whatever is available. |

| Attribute | Detail |
|-----------|--------|
| **Node** | Scrape Barcelona |
| **Type** | `n8n-nodes-idealista-scraper.idealistaScraper` (v1) |
| **Technical Role** | Scrapes Idealista sale listings for the Barcelona market |
| **Configuration** | Operation: `sale`; `numPages`: 3; `locationId`: `0-EU-ES-08-19-001-013`; `locationName`: `Barcelona`; All filter groups left at default (empty = no filters) |
| **Key Expressions** | None |
| **Input** | From: Every Monday 8am |
| **Output** | → Analyze Barcelona |
| **Credential** | Same Apify API credential as Scrape Madrid |
| **Edge Cases / Failures** | Invalid `locationId` format → the scraper may return zero results or an error. The location ID `0-EU-ES-08-19-001-013` is a hierarchical Idealista code — if Idealista changes its internal IDs, this may break. Same Apify credit/timeout risks as Madrid. |

**Important note on parallelism:** Both scrapers run simultaneously because they receive input from the same trigger output. This halves the total scrape time compared to sequential execution.

---

#### Block 3 — Statistical Analysis

**Overview:** Two Code nodes compute identical sets of descriptive statistics for their respective markets. Each receives the raw listing items from its scraper and outputs a single summary item.

**Nodes Involved:** Analyze Madrid, Analyze Barcelona

| Attribute | Detail |
|-----------|--------|
| **Node** | Analyze Madrid |
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Computes aggregate statistics from the Madrid listing data |
| **Configuration** | JavaScript mode; single output item with fields: `market`, `totalListings`, `avgPrice`, `medianPrice`, `minPrice`, `maxPrice`, `avgSize`, `medianSize`, `avgPricePerM2`, `medianPricePerM2`, `avgRooms`, `reportDate` |
| **Key Expressions / Logic** | Extracts `price`, `size`, `priceByArea` or `unitPrice`, `rooms` from each item's JSON. Filters out non-numeric and non-positive values. Computes average (rounded to integer) and median. `avgRooms` is formatted to 1 decimal place. `reportDate` is the ISO date string (YYYY-MM-DD). |
| **Input** | From: Scrape Madrid (array of listing items) |
| **Output** | → Merge Market Data (input index 0) |
| **Edge Cases / Failures** | If zero listings are returned, all numeric outputs default to `0` and `avgRooms` to `'0'`. The `priceByArea` field may not exist on all items — the code falls back to `unitPrice`. If a listing has `price: null` or `price: 0`, it is excluded from calculations (filter: `typeof p === 'number' && p > 0`). Division by zero is avoided by the `arr.length` check. Very large arrays could cause memory pressure in the Code node sandbox. |

| Attribute | Detail |
|-----------|--------|
| **Node** | Analyze Barcelona |
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Computes aggregate statistics from the Barcelona listing data — identical logic to Analyze Madrid |
| **Configuration** | Same JavaScript code structure; outputs `market: 'Barcelona'` instead of `'Madrid'` |
| **Key Expressions / Logic** | Identical to Analyze Madrid — extracts same fields, same statistical functions |
| **Input** | From: Scrape Barcelona (array of listing items) |
| **Output** | → Merge Market Data (input index 1) |
| **Edge Cases / Failures** | Same as Analyze Madrid. Additionally, if the Barcelona scraper returns zero items due to a bad location ID, all stats will be zero, which may produce a misleading report — consider adding a guard downstream. |

**Computed fields per market (output schema):**

| Field | Type | Description |
|-------|------|-------------|
| `market` | string | City name ("Madrid" or "Barcelona") |
| `totalListings` | number | Count of items received from the scraper |
| `avgPrice` | number | Mean price in EUR (rounded) |
| `medianPrice` | number | Median price in EUR (rounded) |
| `minPrice` | number | Minimum price in EUR |
| `maxPrice` | number | Maximum price in EUR |
| `avgSize` | number | Mean size in m² (rounded) |
| `medianSize` | number | Median size in m² (rounded) |
| `avgPricePerM2` | number | Mean price per m² in EUR (rounded) |
| `medianPricePerM2` | number | Median price per m² in EUR (rounded) |
| `avgRooms` | string | Mean number of rooms (1 decimal, as string) |
| `reportDate` | string | ISO date string (YYYY-MM-DD) |

---

#### Block 4 — Data Merging

**Overview:** A Merge node combines the two single-item outputs from the analysis nodes into a unified two-item dataset. This enables downstream nodes to process both markets in a single flow.

**Nodes Involved:** Merge Market Data

| Attribute | Detail |
|-----------|--------|
| **Node** | Merge Market Data |
| **Type** | `n8n-nodes-base.merge` (v3.1) |
| **Technical Role** | Combines two input streams into one output |
| **Configuration** | Mode: Append (default for v3.1 with no parameters set); Input 0 = Analyze Madrid, Input 1 = Analyze Barcelona |
| **Key Expressions** | None |
| **Input** | Input 0: Analyze Madrid; Input 1: Analyze Barcelona |
| **Output** | Two parallel connections: → Build HTML Report, → Log to Market History |
| **Edge Cases / Failures** | If either input branch produces zero items (e.g., one scraper failed entirely), the Merge node will still emit the items from the other branch. However, the Build HTML Report code assumes `markets[0]` exists for the email subject — if both branches are empty, the Code node will throw a reference error. If one branch is empty, the report will only show one market, which may be confusing but won't crash. The Merge node waits for both inputs to arrive before emitting, so if one branch errors upstream, the Merge may wait indefinitely (depending on n8n error handling settings). |

---

#### Block 5 — Report Generation

**Overview:** A Code node takes the merged market statistics and constructs a fully styled HTML email with a comparison table. It also generates the email subject line.

**Nodes Involved:** Build HTML Report

| Attribute | Detail |
|-----------|--------|
| **Node** | Build HTML Report |
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Transforms structured market data into a styled HTML email body and subject line |
| **Configuration** | JavaScript mode; reads all items from `$input.all()`, builds HTML with inline CSS styling including a gradient header, a comparison table, and a footer. Outputs two fields: `subject` and `htmlBody`. |
| **Key Expressions / Logic** | `markets.map(m => m.market).join(' vs ')` constructs the comparison label (e.g., "Madrid vs Barcelona"). Date is formatted as a long English date string. The `fmt()` function localizes numbers. Unicode escapes (`\u00b2`) render the ² symbol for m². The subject line format: `"Weekly Market Report: Madrid vs Barcelona - 2025-01-06"` (uses `reportDate` from the first market). |
| **Input** | From: Merge Market Data (2 items, one per market) |
| **Output** | → Email Report (single item with `subject` and `htmlBody`) |
| **Edge Cases / Failures** | If `markets` array is empty, `markets[0]?.reportDate` safely falls back to `''`, but the `join(' vs ')` produces an empty string, resulting in a malformed subject line. If a market object lacks expected fields (e.g., `avgPrice`), the HTML will render `undefined` — the `fmt` function only handles `number` type; other types are passed through as-is. The HTML is not sanitized — if any field contained HTML-injectable content, it could break the layout (unlikely for numeric data). |

**Output schema:**

| Field | Type | Description |
|-------|------|-------------|
| `subject` | string | Email subject line |
| `htmlBody` | string | Full HTML content of the email report |

---

#### Block 6 — Delivery & Logging

**Overview:** Two parallel output nodes — one sends the HTML report via Gmail, the other appends the raw market statistics to a Google Sheet for historical trend tracking.

**Nodes Involved:** Email Report, Log to Market History

| Attribute | Detail |
|-----------|--------|
| **Node** | Email Report |
| **Type** | `n8n-nodes-base.gmail` (v2.2) |
| **Technical Role** | Sends the HTML market report as an email |
| **Configuration** | `sendTo`: `user@example.com` (placeholder — must be updated); `subject`: `={{ $json.subject }}`; `message`: `={{ $json.htmlBody }}`; Options: default (no custom headers, no attachments). The node sends HTML email because the `message` field contains HTML markup — Gmail node renders it as HTML by default. |
| **Key Expressions** | `$json.subject` and `$json.htmlBody` reference the output of Build HTML Report |
| **Input** | From: Build HTML Report |
| **Output** | None (terminal node) |
| **Credential** | **Gmail OAuth2** credential — must be configured with a Google Cloud project, OAuth consent screen, and authorized redirect URI |
| **Edge Cases / Failures** | OAuth2 token expired → authentication error (n8n auto-refreshes if refresh token is available). Recipient address `user@example.com` is a placeholder and will cause delivery failure if not changed. Gmail daily sending limit (typically 500/day for consumer accounts) should not be an issue for weekly emails. If `htmlBody` is empty or malformed, the email will still be sent but may appear broken. |

| Attribute | Detail |
|-----------|--------|
| **Node** | Log to Market History |
| **Type** | `n8n-nodes-base.googleSheets` (v4.5) |
| **Technical Role** | Appends weekly market statistics to a Google Sheet for longitudinal tracking |
| **Configuration** | Operation: `append`; `documentId` and `sheetName` are set to list mode with empty values — **must be configured** with a real Google Sheet ID and tab name (recommended: "MarketHistory"); No column mapping specified (auto-maps output fields to sheet columns) |
| **Key Expressions** | None |
| **Input** | From: Merge Market Data (2 items — raw market statistics, one per city) |
| **Output** | None (terminal node) |
| **Credential** | **Google Sheets OAuth2** credential — same Google Cloud project as Gmail can be used |
| **Edge Cases / Failures** | If the sheet doesn't exist or the tab name is wrong, the node will error. If the sheet already has data with different column order, the auto-mapping may insert data into wrong columns. If the `documentId` or `sheetName` fields remain empty, the node will fail at runtime. OAuth2 token expiry same risk as Gmail. Rate limiting is unlikely for a single weekly append. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|-----------------|-------------|
| Every Monday 8am | scheduleTrigger (v1.2) | Cron-based workflow trigger (Mon 08:00) | — | Scrape Madrid, Scrape Barcelona | ## Analyze Idealista Real Estate Market Trends and Email Weekly Reports — Every Monday at 8am, this workflow scrapes property listings from multiple Idealista markets, calculates key statistics, builds an HTML comparison report, emails it to you, and logs data to Google Sheets for trend tracking. Idealista has no official API -- this workflow bridges that gap. **This workflow uses the `n8n-nodes-idealista-scraper` community node and requires a self-hosted n8n instance.** |
| Scrape Madrid | idealistaScraper (v1) | Scrape Madrid sale listings from Idealista (3 pages) | Every Monday 8am | Analyze Madrid | ## 1. Scrape Markets — Fetches listings from Madrid and Barcelona in parallel. 3 pages each (~120 properties per city). |
| Scrape Barcelona | idealistaScraper (v1) | Scrape Barcelona sale listings from Idealista (3 pages) | Every Monday 8am | Analyze Barcelona | ## 1. Scrape Markets — Fetches listings from Madrid and Barcelona in parallel. 3 pages each (~120 properties per city). |
| Analyze Madrid | code (v2) | Compute Madrid market statistics (avg/median price, size, rooms, etc.) | Scrape Madrid | Merge Market Data | ## 2. Analyze — Calculates per-market statistics: avg/median price, price per m², inventory count, avg size. |
| Analyze Barcelona | code (v2) | Compute Barcelona market statistics (avg/median price, size, rooms, etc.) | Scrape Barcelona | Merge Market Data | ## 2. Analyze — Calculates per-market statistics: avg/median price, price per m², inventory count, avg size. |
| Merge Market Data | merge (v3.1) | Combine both market analyses into a single dataset | Analyze Madrid (input 0), Analyze Barcelona (input 1) | Build HTML Report, Log to Market History | ## 3. Merge & Report — Combines market data, builds formatted HTML comparison table, emails report via Gmail. |
| Build HTML Report | code (v2) | Generate styled HTML email with market comparison table | Merge Market Data | Email Report | ## 3. Merge & Report — Combines market data, builds formatted HTML comparison table, emails report via Gmail. |
| Email Report | gmail (v2.2) | Send the HTML market report via Gmail | Build HTML Report | — | ## 4. Deliver — Emails the HTML report and logs weekly stats to Google Sheets for trend tracking. |
| Log to Market History | googleSheets (v4.5) | Append weekly market statistics to Google Sheets for trend tracking | Merge Market Data | — | ## 4. Deliver — Emails the HTML report and logs weekly stats to Google Sheets for trend tracking. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your self-hosted n8n instance. Name it "Track Idealista market stats weekly and email Google Sheets reports with Idealista Scraper".

2. **Install the community node:**
   - Go to **Settings → Community Nodes → Install a community node**
   - Enter: `n8n-nodes-idealista-scraper`
   - Confirm installation and wait for the restart prompt.

3. **Create the Apify API credential:**
   - Go to **Credentials → New Credential → Apify API**
   - Paste your personal API token from [https://console.apify.com/account/integrations](https://console.apify.com/account/integrations)
   - Save as "Apify API".

4. **Create the Gmail OAuth2 credential:**
   - In Google Cloud Console, create a project, enable the Gmail API, configure the OAuth consent screen, and create OAuth2 credentials (client ID/secret) with redirect URI matching your n8n instance.
   - In n8n: **Credentials → New Credential → Gmail OAuth2 API**
   - Enter client ID and secret, then complete the OAuth flow to authorize access.
   - Save as "Gmail OAuth2".

5. **Create the Google Sheets OAuth2 credential:**
   - In the same Google Cloud project, enable the Google Sheets API.
   - In n8n: **Credentials → New Credential → Google Sheets OAuth2 API**
   - Enter client ID and secret, complete OAuth flow.
   - Save as "Google Sheets OAuth2".

6. **Prepare the Google Sheet:**
   - Create a new Google Sheet (or use an existing one).
   - Name one tab **"MarketHistory"**.
   - In row 1, add column headers matching the output fields: `market`, `totalListings`, `avgPrice`, `medianPrice`, `minPrice`, `maxPrice`, `avgSize`, `medianSize`, `avgPricePerM2`, `medianPricePerM2`, `avgRooms`, `reportDate`.
   - Note the **Sheet ID** from the URL (the long string between `/d/` and `/edit`).

7. **Add node: "Every Monday 8am"**
   - Type: **Schedule Trigger**
   - Version: 1.2
   - Rule: **Cron Expression** → `0 8 * * 1`

8. **Add node: "Scrape Madrid"**
   - Type: **Idealista Scraper** (community node)
   - Version: 1
   - Credential: Apify API
   - Operation: `sale`
   - Number of Pages: `3`
   - All filter sections: leave at default (empty)
   - No location ID or location name needed (defaults to Madrid)

9. **Add node: "Scrape Barcelona"**
   - Type: **Idealista Scraper** (community node)
   - Version: 1
   - Credential: Apify API (same as step 8)
   - Operation: `sale`
   - Number of Pages: `3`
   - Location ID: `0-EU-ES-08-19-001-013`
   - Location Name: `Barcelona`
   - All filter sections: leave at default (empty)

10. **Connect the trigger to both scrapers in parallel:**
    - From "Every Monday 8am" output → drag to "Scrape Madrid" (main input)
    - From "Every Monday 8am" output → drag to "Scrape Barcelona" (main input)

11. **Add node: "Analyze Madrid"**
    - Type: **Code**
    - Version: 2
    - Mode: JavaScript (run once for all items)
    - Paste the following code:

    ```javascript
    const items = $input.all();
    const prices = items.map(i => i.json.price).filter(p => typeof p === 'number' && p > 0);
    const sizes = items.map(i => i.json.size).filter(s => typeof s === 'number' && s > 0);
    const pricesPerM2 = items.map(i => i.json.priceByArea || i.json.unitPrice).filter(p => typeof p === 'number' && p > 0);
    const rooms = items.map(i => i.json.rooms).filter(r => typeof r === 'number' && r > 0);

    const avg = arr => arr.length ? Math.round(arr.reduce((a, b) => a + b, 0) / arr.length) : 0;
    const median = arr => {
      if (!arr.length) return 0;
      const sorted = [...arr].sort((a, b) => a - b);
      const mid = Math.floor(sorted.length / 2);
      return sorted.length % 2 ? sorted[mid] : Math.round((sorted[mid - 1] + sorted[mid]) / 2);
    };

    return [{
      json: {
        market: 'Madrid',
        totalListings: items.length,
        avgPrice: avg(prices),
        medianPrice: median(prices),
        minPrice: prices.length ? Math.min(...prices) : 0,
        maxPrice: prices.length ? Math.max(...prices) : 0,
        avgSize: avg(sizes),
        medianSize: median(sizes),
        avgPricePerM2: avg(pricesPerM2),
        medianPricePerM2: median(pricesPerM2),
        avgRooms: (rooms.length ? (rooms.reduce((a, b) => a + b, 0) / rooms.length).toFixed(1) : '0'),
        reportDate: new Date().toISOString().split('T')[0]
      }
    }];
    ```

12. **Add node: "Analyze Barcelona"**
    - Type: **Code**
    - Version: 2
    - Mode: JavaScript (run once for all items)
    - Paste the same code as step 11, but change `market: 'Madrid'` to `market: 'Barcelona'`.

13. **Connect scrapers to analyzers:**
    - "Scrape Madrid" → "Analyze Madrid"
    - "Scrape Barcelona" → "Analyze Barcelona"

14. **Add node: "Merge Market Data"**
    - Type: **Merge**
    - Version: 3.1
    - Mode: Append (default — no configuration needed)
    - Connect:
      - "Analyze Madrid" output → Merge input **0**
      - "Analyze Barcelona" output → Merge input **1**

15. **Add node: "Build HTML Report"**
    - Type: **Code**
    - Version: 2
    - Mode: JavaScript (run once for all items)
    - Paste the following code:

    ```javascript
    const markets = $input.all().map(i => i.json);
    const date = new Date().toLocaleDateString('en-US', {
      weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'
    });

    const fmt = n => typeof n === 'number' ? n.toLocaleString('en-US') : n;

    const rows = markets.map(m => `
      <tr>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;font-weight:600;color:#1a202c">${m.market}</td>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;text-align:right">${m.totalListings}</td>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;text-align:right">${fmt(m.avgPrice)} EUR</td>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;text-align:right">${fmt(m.medianPrice)} EUR</td>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;text-align:right">${fmt(m.minPrice)} - ${fmt(m.maxPrice)} EUR</td>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;text-align:right">${m.avgSize} m\u00b2</td>
        <td style="padding:12px 16px;border-bottom:1px solid #e2e8f0;text-align:right">${fmt(m.avgPricePerM2)} EUR/m\u00b2</td>
      </tr>
    `).join('');

    const html = `
    <div style="font-family:'Segoe UI',Arial,sans-serif;max-width:900px;margin:0 auto;padding:20px">
      <div style="background:linear-gradient(135deg,#667eea 0%,#764ba2 100%);padding:30px;border-radius:12px 12px 0 0">
        <h1 style="color:white;margin:0;font-size:24px">Weekly Real Estate Market Report</h1>
        <p style="color:rgba(255,255,255,0.85);margin:8px 0 0 0;font-size:14px">${date} | Source: Idealista.com</p>
      </div>
      <div style="background:white;padding:24px;border:1px solid #e2e8f0;border-top:none;border-radius:0 0 12px 12px">
        <h2 style="color:#2d3748;font-size:18px;margin-top:0">Market Comparison: ${markets.map(m => m.market).join(' vs ')}</h2>
        <table style="width:100%;border-collapse:collapse;margin:16px 0">
          <thead>
            <tr style="background:#f7fafc">
              <th style="padding:12px 16px;text-align:left;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">Market</th>
              <th style="padding:12px 16px;text-align:right;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">Listings</th>
              <th style="padding:12px 16px;text-align:right;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">Avg Price</th>
              <th style="padding:12px 16px;text-align:right;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">Median</th>
              <th style="padding:12px 16px;text-align:right;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">Range</th>
              <th style="padding:12px 16px;text-align:right;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">Avg Size</th>
              <th style="padding:12px 16px;text-align:right;font-size:13px;color:#718096;text-transform:uppercase;letter-spacing:0.5px">EUR/m\u00b2</th>
            </tr>
          </thead>
          <tbody>${rows}</tbody>
        </table>
        <hr style="border:none;border-top:1px solid #e2e8f0;margin:20px 0">
        <p style="color:#a0aec0;font-size:12px;margin:0">Generated automatically by n8n using the Idealista Scraper community node. API-based extraction with 64+ filters across Spain, Italy, and Portugal.</p>
      </div>
    </div>`;

    const subject = `Weekly Market Report: ${markets.map(m => m.market).join(' vs ')} - ${markets[0]?.reportDate || ''}`;

    return [{ json: { subject, htmlBody: html } }];
    ```

16. **Connect Merge to Build HTML Report:**
    - "Merge Market Data" → "Build HTML Report"

17. **Add node: "Email Report"**
    - Type: **Gmail**
    - Version: 2.2
    - Credential: Gmail OAuth2
    - Send To: `user@example.com` — **replace with your actual email address**
    - Subject: `={{ $json.subject }}`
    - Message: `={{ $json.htmlBody }}`
    - Options: leave at defaults

18. **Connect Build HTML Report to Email Report:**
    - "Build HTML Report" → "Email Report"

19. **Add node: "Log to Market History"**
    - Type: **Google Sheets**
    - Version: 4.5
    - Credential: Google Sheets OAuth2
    - Operation: `Append`
    - Document ID: Select your Google Sheet from the list (or paste the Sheet ID from step 6)
    - Sheet Name: Select the **"MarketHistory"** tab
    - No additional column mapping needed (auto-mapped)

20. **Connect Merge to Log to Market History:**
    - "Merge Market Data" → "Log to Market History"
    - This creates a second parallel output from the Merge node alongside Build HTML Report.

21. **Verify all connections match this map:**

    | Source | Target | Notes |
    |--------|--------|-------|
    | Every Monday 8am | Scrape Madrid | Main output |
    | Every Monday 8am | Scrape Barcelona | Main output (parallel) |
    | Scrape Madrid | Analyze Madrid | Main output |
    | Scrape Barcelona | Analyze Barcelona | Main output |
    | Analyze Madrid | Merge Market Data | Input 0 |
    | Analyze Barcelona | Merge Market Data | Input 1 |
    | Merge Market Data | Build HTML Report | Main output |
    | Merge Market Data | Log to Market History | Main output (parallel) |
    | Build HTML Report | Email Report | Main output |

22. **Test the workflow:**
    - Click "Execute Workflow" manually to verify all nodes work.
    - Check your email inbox for the report.
    - Check the Google Sheet for appended rows.
    - Fix any credential or configuration issues.

23. **Activate the workflow:**
    - Toggle the workflow to **Active** in the top-right corner.
    - The workflow will now execute automatically every Monday at 08:00.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow uses the `n8n-nodes-idealista-scraper` community node and requires a self-hosted n8n instance. Community nodes are not supported on n8n Cloud. | Prerequisite for installation |
| Get your Apify API token from the integrations page | [https://console.apify.com/account/integrations](https://console.apify.com/account/integrations) |
| Estimated cost: ~$0.50/week (2 markets × 3 pages × ~40 properties each) on Apify's pricing | Apify pricing consideration |
| To add more cities, duplicate a Scraper + Analysis pair (e.g., Valencia, Rome, Lisbon, Milan) and connect the new Analyze node to the Merge node (adding a third input) | Customization guidance |
| Switch `operation` from `sale` to `rent` on the Scraper nodes to analyze rental markets instead | Customization guidance |
| Add price filters on the Scraper nodes to focus on specific segments (luxury >1M EUR, budget <200K EUR) | Customization guidance |
| Calculate rental yield by scraping both sale and rent for the same area and combining the data | Advanced customization |
| The Idealista Scraper uses API-based extraction (not browser automation), making it resilient to site layout changes — it should not break over time | Reliability note |
| The Gmail node recipient `user@example.com` is a placeholder and must be updated before activation | Critical setup requirement |
| The Google Sheets node `documentId` and `sheetName` are left empty in the template and must be configured with a real Sheet ID and tab name before activation | Critical setup requirement |
| The `locationId` for Barcelona (`0-EU-ES-08-19-001-013`) is a hierarchical Idealista internal code — if Idealista changes its internal ID scheme, this value may need updating | Maintenance note |
| All numeric statistics default to `0` when no listings are returned — this can produce misleading reports; consider adding an error branch or notification if `totalListings === 0` | Edge case warning |
| The Merge node v3.1 in Append mode waits for both inputs — if one branch fails entirely (not just returns empty), the Merge may hang; configure n8n's error handling (Continue On Fail) or add error handling nodes | Reliability note |