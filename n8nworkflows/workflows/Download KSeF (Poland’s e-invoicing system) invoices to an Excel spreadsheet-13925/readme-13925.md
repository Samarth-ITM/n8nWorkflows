Download KSeF (Poland’s e-invoicing system) invoices to an Excel spreadsheet

https://n8nworkflows.xyz/workflows/download-ksef--poland-s-e-invoicing-system--invoices-to-an-excel-spreadsheet-13925


# Download KSeF (Poland’s e-invoicing system) invoices to an Excel spreadsheet

# 1. Workflow Overview

This workflow authenticates against Poland’s **KSeF v2 API**, retrieves invoice metadata for a specified date range, transforms the returned data into flat tabular rows, and exports the result as an **XLSX spreadsheet**.

Typical use cases:
- Downloading invoice metadata you **received** or **issued**
- Building a spreadsheet export for accounting or reporting
- Using KSeF data as a starting point for email, storage, database, or spreadsheet integrations

The workflow is organized into four logical blocks:

## 1.1 Manual Start and Configuration
The workflow starts manually and defines all user-editable runtime settings in a single configuration node: API base URL, NIP, KSeF token, date range, and invoice subject type.

## 1.2 KSeF Authentication Flow
KSeF uses a multi-step authentication model. The workflow retrieves the public certificate, requests a challenge, encrypts the token with RSA-OAEP SHA-256, initializes authentication, waits briefly, checks status, and redeems the temporary token for an access token.

## 1.3 Invoice Query and Pagination
Using the final access token, the workflow queries invoice metadata from KSeF with pagination enabled until all pages are retrieved.

## 1.4 Data Flattening, Spreadsheet Export, and Session Cleanup
The returned invoice metadata is normalized into spreadsheet-friendly columns, written to an XLSX file, and the API session is then closed.

---

# 2. Block-by-Block Analysis

## 2.1 Manual Start and Configuration

### Overview
This block provides the entry point and all required runtime inputs. It centralizes user configuration so the rest of the workflow can reference these values via expressions.

### Nodes Involved
- `When clicking 'Test workflow'`
- `⚙️ Config`

### Node Details

#### When clicking 'Test workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual execution trigger for testing or ad hoc runs.
- **Configuration choices:** No special configuration.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `⚙️ Config`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; only runs when manually triggered.
- **Sub-workflow reference:** None.

#### ⚙️ Config
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines workflow parameters as raw JSON.
- **Configuration choices:** Uses **raw mode** and outputs a single JSON object containing:
  - `baseUrl`: `https://api.ksef.mf.gov.pl/v2`
  - `nip`: user NIP
  - `authToken`: KSeF authorization token
  - `startDate`: ISO 8601 datetime
  - `endDate`: ISO 8601 datetime
  - `subjectType`: `Subject1` or `Subject2`
- **Key expressions or variables used:** These fields are later referenced with expressions such as:
  - `$('⚙️ Config').first().json.baseUrl`
  - `$('⚙️ Config').first().json.nip`
  - `$('⚙️ Config').first().json.authToken`
- **Input and output connections:**  
  - Input: `When clicking 'Test workflow'`
  - Output: `Get Public Key`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged
  - Invalid NIP format
  - Invalid or expired KSeF token
  - Incorrect date format
  - `subjectType` not accepted by KSeF
- **Sub-workflow reference:** None.

---

## 2.2 KSeF Authentication Flow

### Overview
This block performs the full KSeF v2 authentication sequence. It converts the long-lived KSeF authorization token into a temporary auth token, waits for processing, then redeems it into the usable API access token.

### Nodes Involved
- `Get Public Key`
- `Get Challenge`
- `Encrypt Token`
- `Init Auth`
- `Wait 2s`
- `Check Auth Status`
- `Redeem Token`

### Node Details

#### Get Public Key
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches KSeF public certificate data used for token encryption.
- **Configuration choices:** Sends a GET request to:
  - `{{$json.baseUrl}}/security/public-key-certificates`
- **Key expressions or variables used:**
  - `={{ $json.baseUrl }}/security/public-key-certificates`
- **Input and output connections:**  
  - Input: `⚙️ Config`
  - Output: `Get Challenge`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - KSeF endpoint unavailable
  - SSL/network issues
  - Unexpected response structure
- **Sub-workflow reference:** None.

#### Get Challenge
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Requests a challenge and timestamp from KSeF.
- **Configuration choices:** POST request to:
  - `{{ $('⚙️ Config').first().json.baseUrl }}/auth/challenge`
- **Key expressions or variables used:**
  - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/challenge`
- **Input and output connections:**  
  - Input: `Get Public Key`
  - Output: `Encrypt Token`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Endpoint unavailable
  - API contract changes
  - Missing `timestampMs` or `challenge` in response
- **Sub-workflow reference:** None.

#### Encrypt Token
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes Node.js code to build and encrypt the `authToken|timestampMs` payload with the KSeF public certificate.
- **Configuration choices:**
  - Imports `crypto`
  - Reads config from `⚙️ Config`
  - Reads challenge data from current input
  - Reads public keys from `Get Public Key`
  - Selects a certificate intended for `KsefTokenEncryption`
  - Converts a certificate string to PEM if needed
  - Uses RSA OAEP padding with SHA-256
  - Outputs:
    - `challenge`
    - `contextIdentifier.type = "Nip"`
    - `contextIdentifier.value = config.nip`
    - `encryptedToken`
- **Key expressions or variables used:**
  - `$('⚙️ Config').first().json`
  - `$('Get Public Key').all()`
  - `$input.first().json`
- **Input and output connections:**  
  - Input: `Get Challenge`
  - Output: `Init Auth`
- **Version-specific requirements:** Type version `2`.  
  Requires code node execution with access to Node.js built-in `crypto`.
- **Edge cases or potential failure types:**
  - Missing or malformed certificate
  - Certificate array shape differs from expected logic
  - `match(/.{1,64}/g)` could fail if certificate string is empty
  - `authToken` or `timestampMs` missing
  - Runtime restrictions in environments that limit `require('crypto')`
- **Sub-workflow reference:** None.

#### Init Auth
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the encrypted payload to initialize authentication.
- **Configuration choices:**
  - POST to `/auth/ksef-token`
  - Sends JSON body from current item
  - Explicit `Content-Type: application/json`
- **Key expressions or variables used:**
  - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/ksef-token`
  - `={{ JSON.stringify($json) }}`
- **Input and output connections:**  
  - Input: `Encrypt Token`
  - Output: `Wait 2s`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Invalid encrypted payload
  - Invalid NIP/token binding
  - HTTP 4xx/5xx from KSeF
  - Temporary `authenticationToken` returned but auth not yet ready
- **Sub-workflow reference:** None.

#### Wait 2s
- **Type and technical role:** `n8n-nodes-base.code`  
  Introduces a fixed delay before checking auth status.
- **Configuration choices:** Uses `setTimeout` for 2000 ms, then returns input unchanged.
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:**  
  - Input: `Init Auth`
  - Output: `Check Auth Status`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Two seconds may be insufficient under KSeF latency or load
  - No retry loop exists if status is not ready yet
- **Sub-workflow reference:** None.

#### Check Auth Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Polls the auth session status using the temporary authentication token.
- **Configuration choices:**
  - GET to `/auth/{referenceNumber}`
  - Authorization header uses temp token from `Init Auth`
  - Accept header set to `application/json`
- **Key expressions or variables used:**
  - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/{{ $('Init Auth').first().json.referenceNumber }}`
  - `=Bearer {{ $('Init Auth').first().json.authenticationToken.token }}`
- **Input and output connections:**  
  - Input: `Wait 2s`
  - Output: `Redeem Token`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Status still pending after 2 seconds
  - Missing `referenceNumber`
  - Temporary token expired
  - Endpoint errors not explicitly handled
- **Sub-workflow reference:** None.

#### Redeem Token
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Exchanges the temporary auth token for the final access token.
- **Configuration choices:**
  - POST to `/auth/token/redeem`
  - Authorization uses temp token from `Init Auth`
  - Accept header set to `application/json`
- **Key expressions or variables used:**
  - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/token/redeem`
  - `=Bearer {{ $('Init Auth').first().json.authenticationToken.token }}`
- **Input and output connections:**  
  - Input: `Check Auth Status`
  - Output: `Query Invoices`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Attempting redeem before auth is actually ready
  - Expired temporary token
  - KSeF may reject flow if previous status was unsuccessful
- **Sub-workflow reference:** None.

---

## 2.3 Invoice Query and Pagination

### Overview
This block requests invoice metadata from KSeF using the access token and automatically paginates through all available result pages.

### Nodes Involved
- `Query Invoices`
- `Extract Invoices`

### Node Details

#### Query Invoices
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries KSeF invoice metadata with body filters and query pagination controls.
- **Configuration choices:**
  - POST to `/invoices/query/metadata`
  - Sends JSON body:
    - `subjectType` from config
    - `dateRange.dateType = "PermanentStorage"`
    - `from` and `to` from config
  - Query parameters:
    - `pageSize = 250`
    - `sortOrder = Desc`
  - Authorization uses redeemed access token
  - Pagination:
    - `pageOffset = {{$pageCount}}`
    - stops when `{{$response.body.hasMore === false}}`
- **Key expressions or variables used:**
  - `={{ $('⚙️ Config').first().json.baseUrl }}/invoices/query/metadata`
  - `={{ JSON.stringify({ subjectType: $('⚙️ Config').first().json.subjectType, dateRange: { dateType: 'PermanentStorage', from: $('⚙️ Config').first().json.startDate, to: $('⚙️ Config').first().json.endDate } }) }}`
  - `=Bearer {{ $('Redeem Token').first().json.accessToken.token }}`
  - `={{ $pageCount }}`
  - `={{ $response.body.hasMore === false }}`
- **Input and output connections:**  
  - Input: `Redeem Token`
  - Output: `Extract Invoices`
- **Version-specific requirements:** Type version `4.4`.  
  Uses built-in HTTP Request pagination features.
- **Edge cases or potential failure types:**
  - Access token expired or invalid
  - API response lacks `hasMore`
  - `pageOffset` semantics may differ if KSeF changes paging behavior
  - Large result sets could increase runtime
  - Date range may return no data
- **Sub-workflow reference:** None.

#### Extract Invoices
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates invoice arrays from all paginated HTTP responses into a single item stream.
- **Configuration choices:**
  - Iterates over all pages
  - Collects `page.json.invoices`
  - If none found, emits a single marker item:
    - `_noInvoices: true`
    - message
    - count: 0
  - Otherwise emits one item per invoice
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:**  
  - Input: `Query Invoices`
  - Output: `Format for Spreadsheet`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Response structure may not contain `invoices`
  - Empty result set intentionally produces a special record instead of failing
  - Very large datasets may increase memory usage because all pages are collected in memory
- **Sub-workflow reference:** None.

---

## 2.4 Data Flattening, Spreadsheet Export, and Session Cleanup

### Overview
This block converts nested KSeF invoice metadata into flat spreadsheet columns, creates an XLSX binary file, and then closes the active KSeF session.

### Nodes Involved
- `Format for Spreadsheet`
- `Write XLSX`
- `Close Session`

### Node Details

#### Format for Spreadsheet
- **Type and technical role:** `n8n-nodes-base.code`  
  Maps invoice objects into spreadsheet-ready rows with fixed column names.
- **Configuration choices:**
  - If `_noInvoices` marker is present, returns one row with `message: 'No invoices to export'`
  - Otherwise maps each invoice into columns such as:
    - `KSeF Number`
    - `Invoice Number`
    - `Issue Date`
    - `Invoicing Date`
    - `Acquisition Date`
    - `Seller NIP`
    - `Seller Name`
    - `Buyer NIP`
    - `Buyer Name`
    - `Net Amount`
    - `VAT Amount`
    - `Gross Amount`
    - `Currency`
    - `Invoice Type`
    - `Has Attachment`
    - `Self Invoicing`
  - Uses optional chaining and fallback values for missing fields
  - Splits dates at `T` for some date fields
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:**  
  - Input: `Extract Invoices`
  - Output: `Write XLSX`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If invoice schema changes, some columns may be blank
  - Buyer identifier path assumes `buyer.identifier.value`
  - Numeric defaults of `0` may blur distinction between missing and actual zero values
  - Empty output still creates spreadsheet content if `_noInvoices` case occurs
- **Sub-workflow reference:** None.

#### Write XLSX
- **Type and technical role:** `n8n-nodes-base.spreadsheetFile`  
  Converts JSON rows into an XLSX file in binary output.
- **Configuration choices:**
  - Operation: `toFile`
  - File format: `xlsx`
  - File name: `ksef_invoices.xlsx`
  - Sheet name: `Invoices`
  - Header row enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Format for Spreadsheet`
  - Output: `Close Session`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Large datasets may create large binary files
  - If upstream returns only a message row, the file still gets created
  - Binary output must be handled explicitly if saving externally
- **Sub-workflow reference:** None.

#### Close Session
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Deletes the current KSeF auth session after export.
- **Configuration choices:**
  - DELETE to `/auth/sessions/current`
  - Authorization uses final access token
  - `onError` is set to `continueRegularOutput`
- **Key expressions or variables used:**
  - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/sessions/current`
  - `=Bearer {{ $('Redeem Token').first().json.accessToken.token }}`
- **Input and output connections:**  
  - Input: `Write XLSX`
  - Output: none
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Session may already be expired or closed
  - Cleanup failure does not stop normal workflow output because error handling continues
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Test workflow' | Manual Trigger | Manual workflow start |  | ⚙️ Config |  |
| ⚙️ Config | Set | Central runtime configuration for KSeF API, credentials, date range, and subject type | When clicking 'Test workflow' | Get Public Key | ## 🔧 Configuration<br>Edit the JSON below to set your:<br>- **nip** — your 10-digit NIP number<br>- **authToken** — KSeF authorization token<br>- **startDate / endDate** — ISO 8601 format<br>- **subjectType** —<br>  `Subject2` = invoices you **received** (buyer)<br>  `Subject1` = invoices you **issued** (seller)<br><br>Dates use `PermanentStorage` type (when KSeF stored the invoice). |
| Get Public Key | HTTP Request | Fetch KSeF public encryption certificate(s) | ⚙️ Config | Get Challenge | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Get Challenge | HTTP Request | Request KSeF challenge and timestamp | Get Public Key | Encrypt Token | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Encrypt Token | Code | Build and RSA-encrypt KSeF token payload | Get Challenge | Init Auth | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Init Auth | HTTP Request | Initialize KSeF authentication with encrypted token | Encrypt Token | Wait 2s | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Wait 2s | Code | Delay before auth status polling | Init Auth | Check Auth Status | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Check Auth Status | HTTP Request | Poll auth status using temporary token | Wait 2s | Redeem Token | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Redeem Token | HTTP Request | Exchange temporary auth token for final access token | Check Auth Status | Query Invoices | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Query Invoices | HTTP Request | Query paginated invoice metadata | Redeem Token | Extract Invoices | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Extract Invoices | Code | Merge paginated invoice arrays into item stream | Query Invoices | Format for Spreadsheet | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Format for Spreadsheet | Code | Flatten invoice metadata into spreadsheet columns | Extract Invoices | Write XLSX | ## 📊 Output<br>Invoice metadata is flattened into columns:<br>KSeF Number, Invoice Number, Issue Date, Seller/Buyer NIP & Name, Net/VAT/Gross amounts, Currency, Type.<br><br>The XLSX file is available as binary output in the **Write XLSX** node.<br><br>To save to disk, connect a **Write Binary File** node after Write XLSX.<br>To email it, connect a **Send Email** node and attach the binary.<br><br>Swap it to Google Spreadsheet or database of your choice. |
| Write XLSX | Spreadsheet File | Generate XLSX file from flattened rows | Format for Spreadsheet | Close Session | ## 📊 Output<br>Invoice metadata is flattened into columns:<br>KSeF Number, Invoice Number, Issue Date, Seller/Buyer NIP & Name, Net/VAT/Gross amounts, Currency, Type.<br><br>The XLSX file is available as binary output in the **Write XLSX** node.<br><br>To save to disk, connect a **Write Binary File** node after Write XLSX.<br>To email it, connect a **Send Email** node and attach the binary.<br><br>Swap it to Google Spreadsheet or database of your choice. |
| Close Session | HTTP Request | Close current KSeF session after export | Write XLSX |  | ## 📊 Output<br>Invoice metadata is flattened into columns:<br>KSeF Number, Invoice Number, Issue Date, Seller/Buyer NIP & Name, Net/VAT/Gross amounts, Currency, Type.<br><br>The XLSX file is available as binary output in the **Write XLSX** node.<br><br>To save to disk, connect a **Write Binary File** node after Write XLSX.<br>To email it, connect a **Send Email** node and attach the binary.<br><br>Swap it to Google Spreadsheet or database of your choice. |
| Sticky Note | Sticky Note | Visual documentation and quick start guidance |  |  | ## 🇵🇱 KSeF — Download Invoices to Spreadsheet<br><br>Downloads invoice metadata from Poland's **KSeF** (Krajowy System e-Faktur) and exports it as an **XLSX spreadsheet**.<br><br>### Quick Start<br>1. Open the **⚙️ Config** node and fill in your **NIP** and **KSeF token**<br>2. Set the **date range** (startDate / endDate)<br>3. Click **Test workflow**<br>4. Your spreadsheet will appear in the **Write XLSX** node output<br><br>### How to get a KSeF token<br>Generate an authorization token at [ksef.mf.gov.pl](https://ksef.mf.gov.pl) → Log in → Manage tokens.<br>Tokens look like: `YYYYMMDD-XX-XXXXXXXXXX-XXXXXXXXXX-XX\|nip-XXXXXXXXXX\|hash`<br><br>### Need help?<br>KSeF docs: https://www.gov.pl/web/kas/krajowy-system-e-faktur<br><br>Made with ❤️ by [Greg Brzezinka](greg@prosit.no) Need help? [Reach out to me](https://www.linkedin.com/in/brzezinka)! |
| Sticky Note1 | Sticky Note | Visual configuration instructions |  |  | ## 🔧 Configuration<br>Edit the JSON below to set your:<br>- **nip** — your 10-digit NIP number<br>- **authToken** — KSeF authorization token<br>- **startDate / endDate** — ISO 8601 format<br>- **subjectType** —<br>  `Subject2` = invoices you **received** (buyer)<br>  `Subject1` = invoices you **issued** (seller)<br><br>Dates use `PermanentStorage` type (when KSeF stored the invoice). |
| Sticky Note2 | Sticky Note | Visual explanation of KSeF auth sequence |  |  | ## 🔐 Authentication Flow (v2 API)<br>KSeF uses a multi-step auth:<br>1. **Get Public Key** — fetch RSA certificate<br>2. **Get Challenge** — get a challenge + timestamp<br>3. **Encrypt Token** — RSA-OAEP encrypt `token\|timestamp`<br>4. **Init Auth** — submit encrypted token, get temp JWT<br>5. **Wait + Check Status** — poll until auth is ready<br>6. **Redeem Token** — exchange temp JWT for access token<br><br>⚠️ The `authenticationToken` from step 4 is **temporary**!<br>Only the `accessToken` from step 6 works for API calls. |
| Sticky Note3 | Sticky Note | Visual description of spreadsheet output |  |  | ## 📊 Output<br>Invoice metadata is flattened into columns:<br>KSeF Number, Invoice Number, Issue Date, Seller/Buyer NIP & Name, Net/VAT/Gross amounts, Currency, Type.<br><br>The XLSX file is available as binary output in the **Write XLSX** node.<br><br>To save to disk, connect a **Write Binary File** node after Write XLSX.<br>To email it, connect a **Send Email** node and attach the binary.<br><br>Swap it to Google Spreadsheet or database of your choice. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   Name it something like: `KSeF - Download Invoices to Spreadsheet`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Keep default settings.
   - This is the workflow entry point.

3. **Add a Set node named `⚙️ Config`**
   - Node type: **Set**
   - Set mode to **Raw**
   - Paste a JSON object with these keys:
     - `baseUrl`
     - `nip`
     - `authToken`
     - `startDate`
     - `endDate`
     - `subjectType`
   - Example values:
     - `baseUrl`: `https://api.ksef.mf.gov.pl/v2`
     - `nip`: your 10-digit NIP
     - `authToken`: your KSeF token
     - `startDate`: ISO timestamp like `2026-02-01T00:00:00Z`
     - `endDate`: ISO timestamp like `2026-03-06T23:59:59Z`
     - `subjectType`: `Subject2` for received invoices or `Subject1` for issued invoices
   - Connect **Manual Trigger → ⚙️ Config**

4. **Add an HTTP Request node named `Get Public Key`**
   - Method: **GET**
   - URL:
     - `={{ $json.baseUrl }}/security/public-key-certificates`
   - No authentication required.
   - Connect **⚙️ Config → Get Public Key**

5. **Add an HTTP Request node named `Get Challenge`**
   - Method: **POST**
   - URL:
     - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/challenge`
   - No request body needed.
   - Connect **Get Public Key → Get Challenge**

6. **Add a Code node named `Encrypt Token`**
   - Node type: **Code**
   - Language: JavaScript
   - Paste code that:
     - imports Node’s `crypto`
     - reads config from `$('⚙️ Config').first().json`
     - reads `challenge` and `timestampMs` from the input item
     - reads the public certificate(s) from `$('Get Public Key').all()`
     - selects the key whose usage includes `KsefTokenEncryption` if present
     - converts the certificate into PEM format if needed
     - encrypts the plaintext `authToken|timestampMs` using:
       - `RSA_PKCS1_OAEP_PADDING`
       - `oaepHash: 'sha256'`
     - returns JSON with:
       - `challenge`
       - `contextIdentifier: { type: 'Nip', value: config.nip }`
       - `encryptedToken`
   - Connect **Get Challenge → Encrypt Token**

7. **Add an HTTP Request node named `Init Auth`**
   - Method: **POST**
   - URL:
     - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/ksef-token`
   - Send Headers: enabled
   - Header:
     - `Content-Type: application/json`
   - Send Body: enabled
   - Body Content Type: **JSON**
   - JSON body:
     - `={{ JSON.stringify($json) }}`
   - Connect **Encrypt Token → Init Auth**

8. **Add a Code node named `Wait 2s`**
   - Paste JavaScript:
     - wait 2000 ms using `await new Promise(r => setTimeout(r, 2000));`
     - return `$input.all();`
   - Purpose: give KSeF time to prepare auth status.
   - Connect **Init Auth → Wait 2s**

9. **Add an HTTP Request node named `Check Auth Status`**
   - Method: **GET**
   - URL:
     - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/{{ $('Init Auth').first().json.referenceNumber }}`
   - Send Headers: enabled
   - Headers:
     - `Authorization: =Bearer {{ $('Init Auth').first().json.authenticationToken.token }}`
     - `Accept: application/json`
   - Connect **Wait 2s → Check Auth Status**

10. **Add an HTTP Request node named `Redeem Token`**
    - Method: **POST**
    - URL:
      - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/token/redeem`
    - Send Headers: enabled
    - Headers:
      - `Authorization: =Bearer {{ $('Init Auth').first().json.authenticationToken.token }}`
      - `Accept: application/json`
    - No body required.
    - Connect **Check Auth Status → Redeem Token**

11. **Add an HTTP Request node named `Query Invoices`**
    - Method: **POST**
    - URL:
      - `={{ $('⚙️ Config').first().json.baseUrl }}/invoices/query/metadata`
    - Send Headers: enabled
    - Headers:
      - `Content-Type: application/json`
      - `Authorization: =Bearer {{ $('Redeem Token').first().json.accessToken.token }}`
    - Send Query Parameters: enabled
    - Query parameters:
      - `pageSize = 250`
      - `sortOrder = Desc`
    - Send Body: enabled
    - Body Content Type: **JSON**
    - JSON body should contain:
      - `subjectType` from config
      - `dateRange.dateType = PermanentStorage`
      - `dateRange.from = startDate`
      - `dateRange.to = endDate`
    - Use this expression:
      - `={{ JSON.stringify({ subjectType: $('⚙️ Config').first().json.subjectType, dateRange: { dateType: 'PermanentStorage', from: $('⚙️ Config').first().json.startDate, to: $('⚙️ Config').first().json.endDate } }) }}`
    - Enable **pagination**
    - Configure pagination:
      - parameter name: `pageOffset`
      - value: `={{ $pageCount }}`
      - completion expression: `={{ $response.body.hasMore === false }}`
      - completion condition type: **other**
    - Connect **Redeem Token → Query Invoices**

12. **Add a Code node named `Extract Invoices`**
    - JavaScript logic:
      - read all pages using `$input.all()`
      - collect each page’s `json.invoices` array
      - concatenate all invoices into one list
      - if no invoices found, return one item with:
        - `_noInvoices: true`
        - message
        - count: 0
      - otherwise return one item per invoice
    - Connect **Query Invoices → Extract Invoices**

13. **Add a Code node named `Format for Spreadsheet`**
    - JavaScript logic:
      - if input contains only `_noInvoices`, return one row with `message: 'No invoices to export'`
      - otherwise map invoice data into flat columns:
        - `KSeF Number`
        - `Invoice Number`
        - `Issue Date`
        - `Invoicing Date`
        - `Acquisition Date`
        - `Seller NIP`
        - `Seller Name`
        - `Buyer NIP`
        - `Buyer Name`
        - `Net Amount`
        - `VAT Amount`
        - `Gross Amount`
        - `Currency`
        - `Invoice Type`
        - `Has Attachment`
        - `Self Invoicing`
      - use optional chaining and default values
      - for `invoicingDate` and `acquisitionDate`, split on `T` and keep the date part
    - Connect **Extract Invoices → Format for Spreadsheet**

14. **Add a Spreadsheet File node named `Write XLSX`**
    - Operation: **To File**
    - File format: **XLSX**
    - File name: `ksef_invoices.xlsx`
    - Sheet name: `Invoices`
    - Header row: enabled
    - This node will output a binary file.
    - Connect **Format for Spreadsheet → Write XLSX**

15. **Add an HTTP Request node named `Close Session`**
    - Method: **DELETE**
    - URL:
      - `={{ $('⚙️ Config').first().json.baseUrl }}/auth/sessions/current`
    - Send Headers: enabled
    - Header:
      - `Authorization: =Bearer {{ $('Redeem Token').first().json.accessToken.token }}`
    - Set node error handling to **continue on error / continue regular output**
    - This ensures session cleanup failure does not break the workflow.
    - Connect **Write XLSX → Close Session**

16. **Optional: add sticky notes**
    - Add one note with quick-start information and KSeF links
    - Add one near the config node describing `nip`, `authToken`, `startDate`, `endDate`, `subjectType`
    - Add one near the auth chain explaining temp token vs access token
    - Add one near the XLSX output explaining how to save or email the file

17. **Set workflow settings**
    - Timezone: `Europe/Warsaw`
    - Execution order: `v1`

18. **Test the workflow**
    - Open `⚙️ Config`
    - Replace placeholders:
      - `YOUR_NIP_HERE`
      - `YOUR_KSEF_TOKEN_HERE`
    - Set the desired date range
    - Choose `Subject1` or `Subject2`
    - Click **Test workflow**

19. **Validate outputs**
    - `Redeem Token` should return an `accessToken`
    - `Query Invoices` should return paginated results
    - `Write XLSX` should expose binary output containing `ksef_invoices.xlsx`

20. **Optional post-processing**
    - To save locally: connect **Write Binary File** after `Write XLSX`
    - To send by email: connect an email node and attach the binary
    - To write elsewhere: replace or branch after `Format for Spreadsheet` or `Write XLSX`

### Credential configuration
This workflow does **not** use stored n8n credentials by default.  
Authentication is handled directly through the KSeF token placed in the `⚙️ Config` node and then converted into KSeF access tokens through API calls.

### Sub-workflow setup
There are **no sub-workflows** and no Execute Workflow nodes in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate a KSeF authorization token from the KSeF portal under token management. | https://ksef.mf.gov.pl |
| Official KSeF documentation and general information. | https://www.gov.pl/web/kas/krajowy-system-e-faktur |
| Author credit: Greg Brzezinka | `greg@prosit.no` |
| Contact / professional profile of the workflow author. | https://www.linkedin.com/in/brzezinka |
| The workflow exports invoice metadata only, not the full invoice file payload. | General behavior |
| The `authenticationToken` from `Init Auth` is temporary; API calls require the `accessToken` from `Redeem Token`. | Authentication design note |
| The date filter uses `PermanentStorage` as the KSeF date type. | Query design note |
| To persist the generated spreadsheet, attach a `Write Binary File`, email node, cloud storage node, or database flow after `Write XLSX`. | Extension pattern |