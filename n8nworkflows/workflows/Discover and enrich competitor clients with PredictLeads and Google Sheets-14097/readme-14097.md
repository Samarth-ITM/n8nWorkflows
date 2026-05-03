Discover and enrich competitor clients with PredictLeads and Google Sheets

https://n8nworkflows.xyz/workflows/discover-and-enrich-competitor-clients-with-predictleads-and-google-sheets-14097


# Discover and enrich competitor clients with PredictLeads and Google Sheets

## 1. Workflow Overview

This workflow discovers likely client companies of a list of competitors and enriches those discovered companies with company-level metadata before exporting the result to Google Sheets.

Typical use cases:
- Competitive intelligence
- Market mapping
- Prospect discovery
- Building account lists from competitor ecosystems

The workflow is organized into four functional blocks.

### 1.1 Input Reception

The workflow starts manually and reads a list of competitor domains from a Google Sheets tab. Each row is expected to represent one competitor company, identified primarily by its `domain` field.

### 1.2 Client Discovery via PredictLeads Connections API

Each competitor domain is processed one at a time. For each domain, the workflow calls the PredictLeads Connections API and extracts connected companies that appear to be clients, customers, or users.

### 1.3 Client Enrichment via PredictLeads Company API

Each discovered client domain is then processed individually. The workflow calls the PredictLeads Company API to retrieve attributes such as company name, industry, employee count, and geographic location.

### 1.4 Export to Google Sheets

The final enriched records are appended to a Google Sheets output tab, producing a structured table with one row per discovered competitor-client relationship.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Input Reception

**Overview:**  
This block starts the workflow and loads the source list of competitors from Google Sheets. It provides the input dataset used by all downstream discovery and enrichment logic.

**Nodes Involved:**  
- ▶️ Manual Trigger
- 📋 Read Competitors

### Node Details

#### ▶️ Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for on-demand execution.
- **Configuration choices:**  
  No custom configuration. It simply starts the workflow when manually executed.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `📋 Read Competitors`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - No automatic execution; the workflow only runs when manually started.
  - Not suitable as-is for scheduled or event-driven production use.
- **Sub-workflow reference:**  
  None.

#### 📋 Read Competitors
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads competitor records from a Google Sheets document.
- **Configuration choices:**  
  - Uses a specific spreadsheet document ID placeholder: `YOUR_GOOGLE_SHEET_ID_08`
  - Reads from sheet/tab `gid=0`, cached as `Competitors`
  - Uses Google Sheets OAuth2 credentials
  - No additional options configured
- **Key expressions or variables used:**  
  None in parameters, but downstream nodes expect each row to include `domain`.
- **Input and output connections:**  
  - Input: `▶️ Manual Trigger`
  - Output: `🔄 Loop Competitors`
- **Version-specific requirements:**  
  Type version `4.5`; requires a compatible Google Sheets node version in n8n.
- **Edge cases or potential failure types:**  
  - OAuth2 authentication failure
  - Spreadsheet not found or inaccessible
  - Wrong sheet selected
  - Empty sheet
  - Missing `domain` column in input rows
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block: Client Discovery from Competitors

**Overview:**  
This block iterates through competitors, requests connection data from PredictLeads, and extracts connected domains likely to represent clients. It transforms raw API response data into a normalized list of candidate client domains tied to each competitor.

**Nodes Involved:**  
- 🔄 Loop Competitors
- 🔍 Fetch Connections
- ⚙️ Extract Client Domains

### Node Details

#### 🔄 Loop Competitors
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through competitor rows one item at a time.
- **Configuration choices:**  
  Default batching behavior with no custom options shown.
- **Key expressions or variables used:**  
  Downstream code accesses `$('🔄 Loop Competitors').item.json.domain`.
- **Input and output connections:**  
  - Input: `📋 Read Competitors`
  - Secondary loop-back input: `🔄 Loop Clients`
  - Output 1: none configured
  - Output 2: `🔍 Fetch Connections`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If competitor rows are malformed, downstream API requests may fail
  - If `domain` is missing, request URL construction may produce invalid endpoints
  - Loop control depends on downstream return path functioning correctly
- **Sub-workflow reference:**  
  None.

#### 🔍 Fetch Connections
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the PredictLeads Connections API for the current competitor domain.
- **Configuration choices:**  
  - URL expression:  
    `https://predictleads.com/api/v3/companies/{{ $json.domain }}/connections`
  - Sends headers manually:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
  - No extra options such as retry or timeout customization
- **Key expressions or variables used:**  
  - `{{$json.domain}}`
- **Input and output connections:**  
  - Input: `🔄 Loop Competitors`
  - Output: `⚙️ Extract Client Domains`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid or missing domain
  - PredictLeads auth failure due to bad API key/token
  - Rate limiting from PredictLeads
  - 404 if company/domain is not recognized
  - Unexpected response schema
  - Network or timeout issues
- **Sub-workflow reference:**  
  None.

#### ⚙️ Extract Client Domains
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the PredictLeads connections response and emits one item per discovered client domain.
- **Configuration choices:**  
  The JavaScript:
  - Reads all incoming items using `$input.all()`
  - Looks for connections in either:
    - `item.json.data`
    - `item.json.connections`
    - fallback `[]`
  - Reads competitor domain from:
    - `item.json.domain`
    - or `$('🔄 Loop Competitors').item.json.domain`
    - fallback `'unknown'`
  - For each connection, reads attributes from:
    - `conn.attributes`
    - or `conn`
  - Determines relation type from:
    - `connection_type`
    - `relationship_type`
    - `type`
  - Keeps records where relation type contains:
    - `client`
    - `customer`
    - `user`
    - or where relation type is blank
  - Extracts connected domain from:
    - `domain`
    - `company_domain`
    - `connected_company_domain`
  - Emits:
    - `competitor_source`
    - `client_domain`
    - `relationship_type`
  - If no clients are found, emits one marker item:
    - `_no_clients: true`
    - `competitor_source: 'N/A'`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('🔄 Loop Competitors').item.json.domain`
- **Input and output connections:**  
  - Input: `🔍 Fetch Connections`
  - Output: `🔄 Loop Clients`
- **Version-specific requirements:**  
  Type version `2`. Requires Code node JavaScript execution support.
- **Edge cases or potential failure types:**  
  - Response structure may differ from assumed schema
  - If API returns non-array `data`, parsing may behave unexpectedly
  - Empty fallback marker item can create downstream invalid enrichment attempts
  - No deduplication is performed; duplicate domains may pass through
  - Broad inclusion of blank relation types may introduce false positives
- **Sub-workflow reference:**  
  None.

---

## 2.3 Block: Client Company Enrichment

**Overview:**  
This block loops through discovered client domains, enriches each with company data from PredictLeads, and formats the final row structure. It converts discovery output into a report-ready dataset.

**Nodes Involved:**  
- 🔄 Loop Clients
- 🔍 Enrich Client Company
- ⚙️ Format Output Row

### Node Details

#### 🔄 Loop Clients
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over discovered client records one at a time.
- **Configuration choices:**  
  Default options.
- **Key expressions or variables used:**  
  Downstream nodes use:
  - `$('🔄 Loop Clients').item.json.client_domain`
  - `$('🔄 Loop Clients').item.json.competitor_source`
- **Input and output connections:**  
  - Input: `⚙️ Extract Client Domains`
  - Loop-back input: `📊 Write to Google Sheets`
  - Output 1: `🔄 Loop Competitors`
  - Output 2: `🔍 Enrich Client Company`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If the upstream code emits `_no_clients` markers, this loop may still pass items with missing `client_domain`
  - Loop sequencing depends on successful completion of downstream export node
- **Sub-workflow reference:**  
  None.

#### 🔍 Enrich Client Company
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the PredictLeads Company API for each discovered client domain.
- **Configuration choices:**  
  - URL expression:  
    `https://predictleads.com/api/v3/companies/{{ $('🔄 Loop Clients').item.json.client_domain }}`
  - Sends headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `{{ $('🔄 Loop Clients').item.json.client_domain }}`
- **Input and output connections:**  
  - Input: `🔄 Loop Clients`
  - Output: `⚙️ Format Output Row`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing `client_domain`
  - 404 or empty response for unknown domains
  - Auth failure
  - API quota / rate limit
  - Temporary connectivity issues
  - If a `_no_clients` marker item reaches this node, URL may become invalid
- **Sub-workflow reference:**  
  None.

#### ⚙️ Format Output Row
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes the enrichment response into the exact output schema used by Google Sheets.
- **Configuration choices:**  
  The JavaScript:
  - Uses `$input.first()`
  - Reads company data from:
    - `item.json.data?.attributes`
    - or `item.json.attributes`
    - or `item.json`
  - Pulls loop context values:
    - `competitor_source` from `$('🔄 Loop Clients').item.json.competitor_source`
    - `client_domain` from `$('🔄 Loop Clients').item.json.client_domain`
  - Builds final fields:
    - `competitor_source`
    - `client_domain`
    - `client_name`
    - `industry`
    - `employee_count`
    - `location`
  - `location` is constructed from available `city`, `state`, and `country`, joined with commas
  - Uses fallback defaults such as `'Unknown'` or client domain
- **Key expressions or variables used:**  
  - `$input.first()`
  - `$('🔄 Loop Clients').item.json.competitor_source`
  - `$('🔄 Loop Clients').item.json.client_domain`
- **Input and output connections:**  
  - Input: `🔍 Enrich Client Company`
  - Output: `📊 Write to Google Sheets`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If enrichment response lacks expected structure, fallback logic may still work but produce generic values
  - If loop context is missing, output may use `'unknown'`
  - Employee count may remain non-normalized string or number depending on API
  - No validation for malformed location pieces
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block: Data Export & Reporting

**Overview:**  
This block appends the formatted enrichment records to a Google Sheets output tab. It serves as the final persistence layer and closes the inner client-processing loop.

**Nodes Involved:**  
- 📊 Write to Google Sheets

### Node Details

#### 📊 Write to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one formatted row per client to the output sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet document ID placeholder: `YOUR_GOOGLE_SHEET_ID_08`
  - Output sheet/tab: `gid=1354720506`, cached as `Client Discovery`
  - Explicit column mapping enabled
  - Mapped columns:
    - `competitor_source`
    - `client_domain`
    - `client_name`
    - `industry`
    - `employee_count`
    - `location`
  - `attemptToConvertTypes`: false
  - `convertFieldsToString`: false
- **Key expressions or variables used:**  
  - `={{ $json.competitor_source }}`
  - `={{ $json.client_domain }}`
  - `={{ $json.client_name }}`
  - `={{ $json.industry }}`
  - `={{ $json.employee_count }}`
  - `={{ $json.location }}`
- **Input and output connections:**  
  - Input: `⚙️ Format Output Row`
  - Output: `🔄 Loop Clients` (loop continuation)
- **Version-specific requirements:**  
  Type version `4.5`.
- **Edge cases or potential failure types:**  
  - OAuth2 auth failure
  - Sheet not found or wrong tab selected
  - Column mismatch if destination headers differ
  - Append may create duplicate rows on repeated runs
  - Type inconsistencies if output values are mixed strings/numbers
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| About This Workflow | Sticky Note | Workflow description and setup notes |  |  | ABOUT THIS WORKFLOW |
| About This Workflow | Sticky Note | Workflow description and setup notes |  |  | Takes a list of competitor domains, discovers their clients through PredictLeads connections API, enriches each client with company data, and exports everything to Google Sheets. |
| About This Workflow | Sticky Note | Workflow description and setup notes |  |  | Setup: Google Sheet with competitor domains, PredictLeads API, Google Sheets OAuth2. |
| About This Workflow | Sticky Note | Workflow description and setup notes |  |  | Use case: You want to know who your competitor's customers are. Feed in their domain, and this workflow maps out their client relationships with industry, size, and location data. |
| About This Workflow | Sticky Note | Workflow description and setup notes |  |  | PredictLeads API: https://predictleads.com |
| About This Workflow | Sticky Note | Workflow description and setup notes |  |  | Questions: https://www.linkedin.com/in/yaronbeen |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | ## 1️⃣ Trigger & Competitor Source |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | **Nodes:** |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | ▶️ Manual Trigger → 📋 Read Competitors |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | **Description:** |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | The workflow starts with a manual trigger, allowing the user to run the process whenever competitor analysis is needed. |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | It reads a list of **competitor company domains** from a Google Sheets watchlist. |
| ▶️ Manual Trigger | Manual Trigger | Starts the workflow manually |  | 📋 Read Competitors | This sheet acts as the source dataset for the discovery process. Each domain represents a competitor whose client relationships will be analyzed using the PredictLeads API. |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | ## 1️⃣ Trigger & Competitor Source |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | **Nodes:** |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | ▶️ Manual Trigger → 📋 Read Competitors |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | **Description:** |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | The workflow starts with a manual trigger, allowing the user to run the process whenever competitor analysis is needed. |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | It reads a list of **competitor company domains** from a Google Sheets watchlist. |
| 📋 Read Competitors | Google Sheets | Reads competitor domains from source sheet | ▶️ Manual Trigger | 🔄 Loop Competitors | This sheet acts as the source dataset for the discovery process. Each domain represents a competitor whose client relationships will be analyzed using the PredictLeads API. |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 📋 Read Competitors, 🔄 Loop Clients | 🔍 Fetch Connections | ## 2️⃣ Client Discovery from Competitors |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 📋 Read Competitors, 🔄 Loop Clients | 🔍 Fetch Connections | **Nodes:** |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 📋 Read Competitors, 🔄 Loop Clients | 🔍 Fetch Connections | 🔄 Loop Competitors → 🔍 Fetch Connections → ⚙️ Extract Client Domains |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 📋 Read Competitors, 🔄 Loop Clients | 🔍 Fetch Connections | **Description:** |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 📋 Read Competitors, 🔄 Loop Clients | 🔍 Fetch Connections | Each competitor domain is processed individually using a loop. |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 📋 Read Competitors, 🔄 Loop Clients | 🔍 Fetch Connections | The workflow queries the **PredictLeads Connections API** to retrieve company relationships associated with the competitor. |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 🔄 Loop Clients, 📋 Read Competitors | 🔍 Fetch Connections | From the returned connections data, a code node extracts domains that represent **clients, customers, or users** of the competitor. |
| 🔄 Loop Competitors | Split In Batches | Iterates through competitor rows | 🔄 Loop Clients, 📋 Read Competitors | 🔍 Fetch Connections | This step transforms raw connection data into a clean list of **potential client companies linked to each competitor**. |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | ## 2️⃣ Client Discovery from Competitors |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | **Nodes:** |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | 🔄 Loop Competitors → 🔍 Fetch Connections → ⚙️ Extract Client Domains |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | **Description:** |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | Each competitor domain is processed individually using a loop. |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | The workflow queries the **PredictLeads Connections API** to retrieve company relationships associated with the competitor. |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | From the returned connections data, a code node extracts domains that represent **clients, customers, or users** of the competitor. |
| 🔍 Fetch Connections | HTTP Request | Retrieves competitor connection graph from PredictLeads | 🔄 Loop Competitors | ⚙️ Extract Client Domains | This step transforms raw connection data into a clean list of **potential client companies linked to each competitor**. |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | ## 2️⃣ Client Discovery from Competitors |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | **Nodes:** |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | 🔄 Loop Competitors → 🔍 Fetch Connections → ⚙️ Extract Client Domains |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | **Description:** |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | Each competitor domain is processed individually using a loop. |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | The workflow queries the **PredictLeads Connections API** to retrieve company relationships associated with the competitor. |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | From the returned connections data, a code node extracts domains that represent **clients, customers, or users** of the competitor. |
| ⚙️ Extract Client Domains | Code | Filters and emits client domains from connections data | 🔍 Fetch Connections | 🔄 Loop Clients | This step transforms raw connection data into a clean list of **potential client companies linked to each competitor**. |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | ## 3️⃣ Client Company Enrichment |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | **Nodes:** |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | 🔄 Loop Clients → 🔍 Enrich Client Company → ⚙️ Format Output Row |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | **Description:** |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | Each discovered client domain is processed individually. |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | The workflow calls the **PredictLeads Company API** to retrieve detailed company information such as industry, employee count, and location. |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | A code node then formats this enriched data into a structured row that includes: |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | - Competitor source |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | - Client domain |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | - Client company name |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | - Industry |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | - Employee count |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | - Location |
| 🔄 Loop Clients | Split In Batches | Iterates through discovered client domains | ⚙️ Extract Client Domains, 📊 Write to Google Sheets | 🔄 Loop Competitors, 🔍 Enrich Client Company | This creates a clean dataset ready for reporting or further analysis. |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | ## 3️⃣ Client Company Enrichment |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | **Nodes:** |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | 🔄 Loop Clients → 🔍 Enrich Client Company → ⚙️ Format Output Row |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | **Description:** |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | Each discovered client domain is processed individually. |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | The workflow calls the **PredictLeads Company API** to retrieve detailed company information such as industry, employee count, and location. |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | A code node then formats this enriched data into a structured row that includes: |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | - Competitor source |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | - Client domain |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | - Client company name |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | - Industry |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | - Employee count |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | - Location |
| 🔍 Enrich Client Company | HTTP Request | Retrieves company enrichment data for client domain | 🔄 Loop Clients | ⚙️ Format Output Row | This creates a clean dataset ready for reporting or further analysis. |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | ## 3️⃣ Client Company Enrichment |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | **Nodes:** |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | 🔄 Loop Clients → 🔍 Enrich Client Company → ⚙️ Format Output Row |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | **Description:** |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | Each discovered client domain is processed individually. |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | The workflow calls the **PredictLeads Company API** to retrieve detailed company information such as industry, employee count, and location. |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | A code node then formats this enriched data into a structured row that includes: |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | - Competitor source |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | - Client domain |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | - Client company name |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | - Industry |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | - Employee count |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | - Location |
| ⚙️ Format Output Row | Code | Normalizes enrichment data into export schema | 🔍 Enrich Client Company | 📊 Write to Google Sheets | This creates a clean dataset ready for reporting or further analysis. |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | ## 4️⃣ Data Export & Reporting |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | **Nodes:** |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | 📊 Write to Google Sheets |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | **Description:** |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | The enriched client data is appended to a **Google Sheets output table**. |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | Each row represents a discovered client relationship between a competitor and another company, along with key enrichment details. |
| 📊 Write to Google Sheets | Google Sheets | Appends enriched rows to output sheet | ⚙️ Format Output Row | 🔄 Loop Clients | This sheet becomes a structured dataset for **competitor intelligence, market mapping, and potential lead discovery**. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Competitor Client Enrichment & Discovery with PredictLeads`.

2. **Prepare the Google Sheets file**
   - Create one spreadsheet.
   - Add an input tab, for example `Competitors`.
   - Add an output tab, for example `Client Discovery`.
   - In the `Competitors` tab, include at minimum a column named:
     - `domain`
   - In the `Client Discovery` tab, create headers:
     - `competitor_source`
     - `client_domain`
     - `client_name`
     - `industry`
     - `employee_count`
     - `location`

3. **Create Google Sheets credentials in n8n**
   - Add a `Google Sheets OAuth2 API` credential.
   - Authorize the Google account that can access the spreadsheet.
   - Confirm the spreadsheet is visible to that account.

4. **Collect PredictLeads credentials**
   - Obtain:
     - PredictLeads API key
     - PredictLeads API token
   - This workflow uses them directly in HTTP headers rather than in a dedicated credential object.

5. **Add the trigger node**
   - Create a `Manual Trigger` node.
   - Name it `▶️ Manual Trigger`.

6. **Add the competitor input node**
   - Create a `Google Sheets` node.
   - Name it `📋 Read Competitors`.
   - Connect `▶️ Manual Trigger` → `📋 Read Competitors`.
   - Configure it to read from the spreadsheet document.
   - Select the input tab containing competitor domains.
   - Attach the Google Sheets OAuth2 credential.

7. **Add the outer loop**
   - Create a `Split In Batches` node.
   - Name it `🔄 Loop Competitors`.
   - Connect `📋 Read Competitors` → `🔄 Loop Competitors`.
   - Leave default options unless you want to customize batching behavior.

8. **Add the PredictLeads connections request**
   - Create an `HTTP Request` node.
   - Name it `🔍 Fetch Connections`.
   - Connect the second/main iteration output of `🔄 Loop Competitors` to `🔍 Fetch Connections`.
   - Set method to `GET` if needed by your n8n version.
   - Set URL to:
     `=https://predictleads.com/api/v3/companies/{{ $json.domain }}/connections`
   - Enable custom headers.
   - Add:
     - `X-Api-Key: YOUR_PREDICTLEADS_API_KEY`
     - `X-Api-Token: YOUR_PREDICTLEADS_API_TOKEN`
     - `Content-Type: application/json`

9. **Add the client extraction code node**
   - Create a `Code` node.
   - Name it `⚙️ Extract Client Domains`.
   - Connect `🔍 Fetch Connections` → `⚙️ Extract Client Domains`.
   - Paste this logic in JavaScript form:
     - Read input items
     - Pull connections from `item.json.data` or `item.json.connections`
     - For each connection, inspect `connection_type`, `relationship_type`, or `type`
     - Keep relationships containing `client`, `customer`, `user`, or blank type
     - Extract a domain from `domain`, `company_domain`, or `connected_company_domain`
     - Output one item per client with:
       - `competitor_source`
       - `client_domain`
       - `relationship_type`
     - If none found, return an item containing `_no_clients: true`
   - If rebuilding exactly, use the same fallback behavior for unknown values.

10. **Add the inner loop**
    - Create another `Split In Batches` node.
    - Name it `🔄 Loop Clients`.
    - Connect `⚙️ Extract Client Domains` → `🔄 Loop Clients`.

11. **Add the client company enrichment request**
    - Create an `HTTP Request` node.
    - Name it `🔍 Enrich Client Company`.
    - Connect the main iteration output of `🔄 Loop Clients` → `🔍 Enrich Client Company`.
    - Set URL to:
      `=https://predictleads.com/api/v3/companies/{{ $('🔄 Loop Clients').item.json.client_domain }}`
    - Add the same headers:
      - `X-Api-Key`
      - `X-Api-Token`
      - `Content-Type: application/json`

12. **Add the formatting code node**
    - Create a `Code` node.
    - Name it `⚙️ Format Output Row`.
    - Connect `🔍 Enrich Client Company` → `⚙️ Format Output Row`.
    - Implement JavaScript that:
      - Reads first input item
      - Extracts company data from `data.attributes`, `attributes`, or root JSON
      - Reads:
        - `competitor_source` from `$('🔄 Loop Clients').item.json.competitor_source`
        - `client_domain` from `$('🔄 Loop Clients').item.json.client_domain`
      - Outputs:
        - `competitor_source`
        - `client_domain`
        - `client_name`
        - `industry`
        - `employee_count`
        - `location`
      - Builds `location` from `city`, `state`, and `country`
      - Uses fallback values like `Unknown`

13. **Add the Google Sheets output node**
    - Create another `Google Sheets` node.
    - Name it `📊 Write to Google Sheets`.
    - Connect `⚙️ Format Output Row` → `📊 Write to Google Sheets`.
    - Set operation to `Append`.
    - Select the same spreadsheet document.
    - Select the output tab `Client Discovery`.
    - Attach the same Google Sheets OAuth2 credential.
    - Enable explicit field mapping.
    - Map:
      - `competitor_source` → `{{ $json.competitor_source }}`
      - `client_domain` → `{{ $json.client_domain }}`
      - `client_name` → `{{ $json.client_name }}`
      - `industry` → `{{ $json.industry }}`
      - `employee_count` → `{{ $json.employee_count }}`
      - `location` → `{{ $json.location }}`

14. **Close the inner client loop**
    - Connect `📊 Write to Google Sheets` back to `🔄 Loop Clients` loop continuation input.
   - This allows the workflow to continue processing all discovered clients for the current competitor.

15. **Close the outer competitor loop**
    - Connect the completion output of `🔄 Loop Clients` to `🔄 Loop Competitors`.
   - This causes the workflow to resume with the next competitor after all current competitor clients have been processed.

16. **Validate the loop wiring carefully**
   - Outer loop flow:
     - `📋 Read Competitors` → `🔄 Loop Competitors` → `🔍 Fetch Connections` → `⚙️ Extract Client Domains` → `🔄 Loop Clients`
   - Inner loop flow:
     - `🔄 Loop Clients` → `🔍 Enrich Client Company` → `⚙️ Format Output Row` → `📊 Write to Google Sheets` → back to `🔄 Loop Clients`
   - Loop completion:
     - `🔄 Loop Clients` completion output → back to `🔄 Loop Competitors`

17. **Optionally add sticky notes**
   - Add notes for:
     - input/source sheet purpose
     - discovery logic
     - enrichment logic
     - output/reporting purpose
     - workflow setup details and links

18. **Run a test**
   - Put a few known domains in the `Competitors` tab.
   - Execute manually.
   - Confirm:
     - the connections request returns data
     - client domains are extracted
     - enrichment returns company metadata
     - rows append correctly in the output tab

19. **Recommended hardening improvements**
   - Filter out `_no_clients` items before enrichment
   - Add deduplication for repeated client domains
   - Add retry or continue-on-fail handling for HTTP nodes
   - Add validation for missing input `domain`
   - Store PredictLeads secrets in credentials or environment variables instead of literal values
   - Optionally clear or timestamp output rows to track separate runs

20. **No sub-workflow setup is required**
   - This workflow contains no Execute Workflow node and no called sub-workflow.
   - It has only one entry point: `▶️ Manual Trigger`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PredictLeads API | https://predictleads.com |
| Questions / author contact | https://www.linkedin.com/in/yaronbeen |
| Workflow purpose | Takes a list of competitor domains, discovers their clients through PredictLeads connections API, enriches each client with company data, and exports everything to Google Sheets. |
| Required setup | Google Sheet with competitor domains, PredictLeads API access, Google Sheets OAuth2 credentials. |
| Primary use case | Identify who a competitor’s customers may be by mapping domain-based relationships and enriching discovered companies with industry, size, and location data. |

### Additional implementation notes
- The workflow assumes the source sheet contains a `domain` column.
- The workflow does not currently skip `_no_clients` markers before calling the enrichment API.
- The workflow appends data and does not prevent duplicates across runs.
- There is no built-in rate limit handling for PredictLeads.
- There are no sub-workflows in this design.