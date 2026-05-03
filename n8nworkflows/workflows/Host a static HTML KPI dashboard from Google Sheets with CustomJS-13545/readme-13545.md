Host a static HTML KPI dashboard from Google Sheets with CustomJS

https://n8nworkflows.xyz/workflows/host-a-static-html-kpi-dashboard-from-google-sheets-with-customjs-13545


# Host a static HTML KPI dashboard from Google Sheets with CustomJS

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Generate a KPI dashboard from a Google Sheet, render it into a static HTML page (with Chart.js), publish/update that page on **CustomJS**, and then generate a **PDF containing a QR code** that points to the hosted dashboard URL.

**Typical use cases**
- Weekly KPI reporting dashboards shared via a single live URL
- Lightweight “BI-style” reporting without a full web app
- Automated publication + distribution artifact (QR PDF)

### Logical Blocks
1.1 **Execution & Scheduling**  
- Entry points: manual execution and time-based schedule.

1.2 **Data Retrieval & Normalization (Google Sheets → JSON)**  
- Download public sheet as CSV, extract rows, aggregate into a single JSON payload.

1.3 **HTML Dashboard Build (JSON → HTML)**  
- Inject structured data into a full HTML template that builds KPI cards, a table, and charts.

1.4 **CustomJS Hosting (Update-or-Create Page)**  
- List pages, find the page named `Test`, update if exists; otherwise create.

1.5 **QR Code PDF Generation**  
- Convert a small HTML document that renders a QR code (pointing to `htmlFileUrl`) into a PDF.

---

## 2. Block-by-Block Analysis

### 2.1 Execution & Scheduling
**Overview:** Starts the workflow either manually or on a schedule. Both triggers feed into the same data-fetching path.

**Nodes involved**
- When clicking ‘Execute workflow’
- Schedule Trigger

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; ad-hoc execution entry point.
- **Key config:** No parameters.
- **Outputs:** Triggers `Get Data From Sheet`.
- **Edge cases:** None (except user permissions to run workflows).

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger; automated periodic execution.
- **Key config:** `rule.interval` is present but effectively **not configured** (empty object). In n8n this typically means the schedule may be invalid or default behavior is unclear.
- **Outputs:** Triggers `Get Data From Sheet`.
- **Edge cases / failures:**
  - Misconfigured schedule may prevent runs entirely.
  - Timezone considerations (instance timezone vs desired reporting timezone).

---

### 2.2 Data Retrieval & Normalization (Google Sheets → JSON)
**Overview:** Downloads a Google Sheet as CSV, parses it into items, then aggregates all rows into a single array under one item for easier HTML injection.

**Nodes involved**
- Get Data From Sheet
- Extract from File
- Aggregate

#### Node: Get Data From Sheet
- **Type / role:** HTTP Request; fetches Google Sheets CSV export.
- **Key config choices:**
  - URL uses Google Visualization CSV export:  
    `https://docs.google.com/spreadsheets/d/<id>/gviz/tq?tqx=out:csv&sheet=Sheet1`
  - Response format: **File** (binary), not JSON/text.
- **Input:** From either trigger.
- **Output:** Binary file containing CSV.
- **Edge cases / failures:**
  - Sheet not public / permission denied (returns HTML login page instead of CSV).
  - Sheet name mismatch (`Sheet1`) → may return error/empty.
  - Network/timeouts.
  - If Google returns non-CSV content, downstream extraction may fail.

#### Node: Extract from File
- **Type / role:** Extract From File; parses the downloaded file into structured items.
- **Key config:** Defaults; relies on auto-detection and/or binary input format.
- **Input:** Binary CSV from `Get Data From Sheet`.
- **Output:** One item per CSV row (typically key-value pairs by header).
- **Edge cases / failures:**
  - If the prior node returned HTML instead of CSV, parsing fails.
  - CSV headers with unexpected names or extra spaces affect downstream field lookups (e.g., `"Demo Booked"` must match exactly).

#### Node: Aggregate
- **Type / role:** Aggregate; combines all incoming items into a single item payload.
- **Key config:** `aggregateAllItemData` (collect all items).
- **Input:** Parsed rows from `Extract from File`.
- **Output:** A single item that contains an array of all rows (used later as `$json.data`).
- **Edge cases / failures:**
  - Empty input (no rows) → HTML may show empty charts/tables; calculations may produce `NaN` if not guarded.

**Sticky note coverage:** “## Collect sheet data and convert to json” (applies to the nodes in this block)

---

### 2.3 HTML Dashboard Build (JSON → HTML)
**Overview:** Produces a full static HTML dashboard by injecting the aggregated JSON into an HTML template. Charts are rendered client-side via Chart.js.

**Nodes involved**
- HTML

#### Node: HTML
- **Type / role:** HTML node; generates an `html` field from a provided template.
- **Key config choices:**
  - Full HTML document with inline CSS and JS.
  - Loads Chart.js from CDN: `https://cdn.jsdelivr.net/npm/chart.js`
  - Injects data with an n8n expression inside script:
    - `data: {{ JSON.stringify($json.data) }}`
- **Input:** Single aggregated item from `Aggregate` containing `data` array.
- **Output:** Item containing `html` (and likely passes through existing fields depending on node behavior).
- **Important internal logic / variables:**
  - Builds `report = { reportTitle, period, data }`.
  - Computes totals via `reduce`. **Potential bug due to operator precedence**:
    - Example line: `visitors: a.visitors + c["Visitors"] || 0`
    - If `c["Visitors"]` is undefined, `a.visitors + undefined` becomes `NaN`, and `NaN || 0` evaluates to `0` (because `NaN` is falsy), which **resets** rather than accumulating safely.
    - Safer pattern: `visitors: a.visitors + (Number(c["Visitors"]) || 0)`
  - Calculates rates:
    - `leadRate = ((totals.leads / totals.visitors)*100).toFixed(1)`
    - If `totals.visitors` is 0 → `Infinity` or `NaN`.
- **Connections:** Outputs to `Get HTML Pages`.
- **Edge cases / failures:**
  - Missing columns: `"Visitors"`, `"Leads"`, `"Demo Booked"`, `"Proposal Sent"`, `"Won"`, `"Channel"` → charts/tables break or show “undefined”.
  - Non-numeric strings in numeric columns → NaN.
  - Large datasets may produce heavy HTML and slow rendering.

**Sticky note coverage:** “## Build and publish HTML” (this block + next hosting block)

---

### 2.4 CustomJS Hosting (Update-or-Create Page)
**Overview:** Fetches all existing CustomJS pages, filters for a page named `Test`, then updates it if found; otherwise uploads a new page. Both branches proceed to QR PDF generation.

**Nodes involved**
- Get HTML Pages
- Filter
- If
- Update existing HTML Page
- Upload new HTML Page

#### Node: Get HTML Pages
- **Type / role:** CustomJS node (`@custom-js/n8n-nodes-pdf-toolkit-v2.pdfToolkit`); lists hosted pages.
- **Key config:** Resource `page`, operation `getAll`.
- **Credentials:** `CustomJS account` (API key-based).
- **Input:** From `HTML`.
- **Output:** One item per page (typical fields include name, ids, workspaceId, etc.).
- **Edge cases / failures:**
  - Auth failure (invalid/expired API key).
  - API downtime / rate limits.
  - Empty list if workspace has no pages.

#### Node: Filter
- **Type / role:** Filter; keeps only pages whose name equals `Test`.
- **Key config:**
  - Condition: `{{$json.name}}` **equals** `Test`
  - `alwaysOutputData: true` (important)
- **Input:** Items (pages) from `Get HTML Pages`.
- **Output:** Matching page items; if none match, still outputs (because alwaysOutputData is on), but content may be empty/unchanged depending on n8n’s filter behavior.
- **Edge cases / failures:**
  - If CustomJS page objects don’t have `name` exactly, nothing matches.
  - Multiple pages named `Test`: multiple items pass through (affects update branch).

#### Node: If
- **Type / role:** IF; decides between update vs create.
- **Key config:**
  - Condition: `{{$json.workspaceId}}` **exists**
  - True → Update existing HTML Page
  - False → Upload new HTML Page
- **Input:** From `Filter`.
- **Output:** Branching.
- **Edge cases / failures:**
  - If `Filter` outputs no valid page item but still produces an item shell, `workspaceId` may not exist → create branch.
  - If multiple `Test` pages exist, true branch may run multiple times (multiple updates).

#### Node: Update existing HTML Page
- **Type / role:** CustomJS page update.
- **Key config:**
  - Operation: `update`
  - `pageId`: `={{ $json.pageId }}`
  - `pageName`: `Test`
  - `htmlContent`: `={{ $('HTML').item.json.html }}`
    - Note: explicitly references the `HTML` node’s output.
- **Input:** Matching page item from `If` true branch.
- **Output:** Updated page object, expected to include a hosted URL (commonly `htmlFileUrl`).
- **Edge cases / failures:**
  - Missing `pageId` in returned objects → update fails.
  - HTML too large or invalid for service constraints.
  - Cross-item reference `$('HTML').item...` assumes the correct pairing; with multiple items, ensure the right item is referenced.

#### Node: Upload new HTML Page
- **Type / role:** CustomJS page creation.
- **Key config:**
  - Resource: `page`
  - `pageName`: `Test`
  - `htmlContent`: `={{ $json.html }}`
- **Input:** Comes from `If` false branch; however, note that `$json.html` is not guaranteed to exist at that point unless the incoming item still contains `html`. Depending on how `Get HTML Pages`/`Filter`/`If` pass through data, this can be fragile.
  - In contrast, the update node safely references `$('HTML').item.json.html`.
- **Output:** Created page object, expected to include `htmlFileUrl`.
- **Edge cases / failures:**
  - **Most likely bug:** `$json.html` may be undefined here. Safer to use `={{ $('HTML').item.json.html }}` (same as update).
  - Name conflicts: if service disallows duplicates, create may fail.
  - Auth/validation errors.

---

### 2.5 QR Code PDF Generation
**Overview:** Builds a small HTML snippet that renders a QR code pointing to the hosted dashboard URL (`htmlFileUrl`), then converts that HTML to a PDF via CustomJS.

**Nodes involved**
- Convert HTML to PDF

#### Node: Convert HTML to PDF
- **Type / role:** CustomJS HTML-to-PDF conversion.
- **Key config choices:**
  - Operation: `htmlToPdf`
  - Uses an inline HTML document that loads QRCode.js:
    - `https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js`
  - QR code text source: `const url = "{{ $json.htmlFileUrl }}";`
  - `pdfHeightMm: 120` (custom page height)
- **Input:** Output from either `Update existing HTML Page` or `Upload new HTML Page`.
- **Output:** PDF (likely binary) generated by CustomJS.
- **Edge cases / failures:**
  - If `htmlFileUrl` is missing from the previous node output, QR encodes `"undefined"` or empty.
  - External CDN blocked (network policy) could prevent QR rendering in HTML-to-PDF engine (depends if converter loads external resources).
  - Service limits on conversion time/size.

**Sticky note coverage:** “## Generate QR code” (applies to this node)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual entry point | — | Get Data From Sheet | ## Collect sheet data and convert to json |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point | — | Get Data From Sheet | ## Collect sheet data and convert to json |
| Get Data From Sheet | n8n-nodes-base.httpRequest | Download Google Sheet as CSV (binary) | Manual Trigger, Schedule Trigger | Extract from File | ## Collect sheet data and convert to json |
| Extract from File | n8n-nodes-base.extractFromFile | Parse CSV into items | Get Data From Sheet | Aggregate | ## Collect sheet data and convert to json |
| Aggregate | n8n-nodes-base.aggregate | Combine all rows into one item (`data` array) | Extract from File | HTML | ## Collect sheet data and convert to json |
| HTML | n8n-nodes-base.html | Render dashboard HTML with embedded JSON + Chart.js | Aggregate | Get HTML Pages | ## Build and publish HTML |
| Get HTML Pages | @custom-js/n8n-nodes-pdf-toolkit-v2.pdfToolkit | List CustomJS pages | HTML | Filter | ## Build and publish HTML |
| Filter | n8n-nodes-base.filter | Select page(s) named `Test` | Get HTML Pages | If | ## Build and publish HTML |
| If | n8n-nodes-base.if | Decide update vs create | Filter | Update existing HTML Page (true), Upload new HTML Page (false) | ## Build and publish HTML |
| Update existing HTML Page | @custom-js/n8n-nodes-pdf-toolkit-v2.pdfToolkit | Update existing hosted page | If (true) | Convert HTML to PDF | ## Build and publish HTML |
| Upload new HTML Page | @custom-js/n8n-nodes-pdf-toolkit-v2.pdfToolkit | Create hosted page | If (false) | Convert HTML to PDF | ## Build and publish HTML |
| Convert HTML to PDF | @custom-js/n8n-nodes-pdf-toolkit-v2.pdfToolkit | Generate QR PDF pointing to hosted HTML URL | Update existing HTML Page, Upload new HTML Page | — | ## Generate QR code |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/annotation | — | — | # Hosting Static HTML KPI Dashboard with CustomJS  \n[KPI Google Sheet](https://docs.google.com/spreadsheets/d/10mj6hngkPTg1_A6bVNn8QfKujap6j-Lrt_LwAkY2d7Q/edit?usp=sharing)  \nThis workflow automatically generates a weekly KPI dashboard from Google Sheets data and hosts it as a live static HTML page using [CustomJS](https://www.customjs.space).\n\n## Setup\n- Requires a **self-hosted n8n instance** and a **CustomJS API key**.\n- Google Sheets should be **publicly accessible** or shared with a service account for n8n access.\n- Optional: Additional JSON/CSV sources can be integrated.\n\n## How it works\n1 Manual Trigger  \n2 Load Data from Google Sheet  \n3 Prepare Structured JSON  \n4 Generate HTML Dashboard  \n5 Host Static HTML  \n6 Optional Enhancements |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/annotation | — | — | ## Collect sheet data and convert to json |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/annotation | — | — | ## Build and publish HTML |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/annotation | — | — | ## Generate QR code |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named:  
   **“Host a static HTML KPI dashboard from Google Sheets with CustomJS”**

2) **Add Trigger nodes**
   1. Add **Manual Trigger** node named **“When clicking ‘Execute workflow’”**.
   2. Add **Schedule Trigger** node named **“Schedule Trigger”**.
      - Configure an actual interval (e.g., every week on Monday 08:00). The provided workflow’s schedule rule is not effectively set.

3) **Add “Get Data From Sheet” (HTTP Request)**
   - Node type: **HTTP Request**
   - Name: **Get Data From Sheet**
   - Method: GET
   - URL:  
     `https://docs.google.com/spreadsheets/d/10mj6hngkPTg1_A6bVNn8QfKujap6j-Lrt_LwAkY2d7Q/gviz/tq?tqx=out:csv&sheet=Sheet1`
   - Response: set **Response Format = File** (binary)
   - Connect:
     - Manual Trigger → Get Data From Sheet
     - Schedule Trigger → Get Data From Sheet

4) **Add “Extract from File”**
   - Node type: **Extract From File**
   - Name: **Extract from File**
   - Keep default extraction options (ensure it parses CSV from the binary property).
   - Connect: Get Data From Sheet → Extract from File

5) **Add “Aggregate”**
   - Node type: **Aggregate**
   - Name: **Aggregate**
   - Operation/mode: **Aggregate All Item Data** (collect all incoming rows into one object containing `data`)
   - Connect: Extract from File → Aggregate

6) **Add “HTML” (HTML generation)**
   - Node type: **HTML**
   - Name: **HTML**
   - Paste the dashboard HTML template.
   - Ensure the injected data line uses the aggregated array:
     - `data: {{ JSON.stringify($json.data) }}`
   - Connect: Aggregate → HTML

7) **Set up CustomJS credentials**
   - Install/enable the **CustomJS n8n community node** package used here: `@custom-js/n8n-nodes-pdf-toolkit-v2` (if not already installed on self-hosted n8n).
   - Create credentials: **CustomJS API** (API key from CustomJS).
   - Select that credential in each CustomJS node below.

8) **Add “Get HTML Pages”**
   - Node type: **CustomJS (pdfToolkit)**  
   - Name: **Get HTML Pages**
   - Resource: `page`
   - Operation: `getAll`
   - Connect: HTML → Get HTML Pages

9) **Add “Filter”**
   - Node type: **Filter**
   - Name: **Filter**
   - Condition: `{{$json.name}}` **equals** `Test`
   - Enable **Always Output Data** (to allow the flow to continue even if no match is found).
   - Connect: Get HTML Pages → Filter

10) **Add “If” (update vs create)**
   - Node type: **If**
   - Name: **If**
   - Condition: `{{$json.workspaceId}}` **exists**
   - Connect: Filter → If

11) **Add “Update existing HTML Page”**
   - Node type: **CustomJS (pdfToolkit)**
   - Name: **Update existing HTML Page**
   - Resource: `page`
   - Operation: `update`
   - `pageId`: `={{ $json.pageId }}`
   - `pageName`: `Test`
   - `htmlContent`: `={{ $('HTML').item.json.html }}`
   - Connect: If (true output) → Update existing HTML Page

12) **Add “Upload new HTML Page”**
   - Node type: **CustomJS (pdfToolkit)**
   - Name: **Upload new HTML Page**
   - Resource: `page`
   - Operation: create/upload (node shows resource `page` without explicit operation; configure to “create” if required by your node UI)
   - `pageName`: `Test`
   - **Recommended** `htmlContent`: `={{ $('HTML').item.json.html }}`  
     (more robust than `={{ $json.html }}` in this branching context)
   - Connect: If (false output) → Upload new HTML Page

13) **Add “Convert HTML to PDF” (QR code)**
   - Node type: **CustomJS (pdfToolkit)**
   - Name: **Convert HTML to PDF**
   - Operation: `htmlToPdf`
   - HTML: paste the QR HTML, ensuring it references the hosted page URL:
     - `const url = "{{ $json.htmlFileUrl }}";`
   - Set `pdfHeightMm` to `120` (as in workflow)
   - Connect:
     - Update existing HTML Page → Convert HTML to PDF
     - Upload new HTML Page → Convert HTML to PDF

14) **(Optional) Add output handling**
   - If you want to store/send the PDF, add nodes after “Convert HTML to PDF” (e.g., Email, Google Drive, S3). The provided workflow ends after PDF creation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| KPI Google Sheet (public example) | https://docs.google.com/spreadsheets/d/10mj6hngkPTg1_A6bVNn8QfKujap6j-Lrt_LwAkY2d7Q/edit?usp=sharing |
| CustomJS hosting platform | https://www.customjs.space |
| Requires self-hosted n8n + CustomJS API key | Mentioned in sticky note setup section |
| Data columns must match exactly (`Visitors`, `Leads`, `Demo Booked`, `Proposal Sent`, `Won`, `Channel`) | Used in HTML template lookups |
| Consider hardening totals calculation to avoid NaN/Infinity | Current reduce logic and rate math can break on missing/zero values |

