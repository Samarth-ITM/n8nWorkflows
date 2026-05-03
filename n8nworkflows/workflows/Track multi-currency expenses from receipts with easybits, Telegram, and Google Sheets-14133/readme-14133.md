Track multi-currency expenses from receipts with easybits, Telegram, and Google Sheets

https://n8nworkflows.xyz/workflows/track-multi-currency-expenses-from-receipts-with-easybits--telegram--and-google-sheets-14133


# Track multi-currency expenses from receipts with easybits, Telegram, and Google Sheets

# 1. Workflow Overview

This workflow automates receipt-based expense tracking from Telegram into Google Sheets, with live currency conversion to EUR.

A user sends a receipt photo to a Telegram bot. The workflow downloads the image, converts it into a format accepted by an easybits extraction pipeline, sends it to easybits for structured field extraction, normalizes the returned values, fetches an exchange rate with a fallback provider, computes the EUR equivalent, and appends the result to a Google Sheet.

## 1.1 Input Reception and File Preparation
This block receives an image from Telegram, extracts the binary content, and converts it into a Base64 data URL suitable for API submission.

## 1.2 AI Extraction via easybits
This block sends the receipt image to an easybits Extractor pipeline and receives structured invoice data.

## 1.3 Data Normalization
This block parses the extracted payload and applies defaults when the AI output is incomplete or inconsistently typed.

## 1.4 Exchange Rate Retrieval with Fallback
This block requests a currency-to-EUR exchange rate from a primary endpoint and falls back to a secondary endpoint if the primary response indicates an error.

## 1.5 EUR Conversion and Spreadsheet Logging
This block computes the converted EUR amount and appends the invoice record to Google Sheets.

## 1.6 Embedded Documentation / Visual Guidance
The workflow also contains multiple sticky notes that explain setup, assumptions, and integration requirements directly in the canvas.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and File Preparation

### Overview
This block listens for Telegram messages containing receipt photos, downloads the file, extracts the binary into a property, and reformats it as a Base64 data URL. The output is prepared specifically for the easybits HTTP API.

### Nodes Involved
- Telegram: Receipt Photo
- Extract from File
- Edit Fields

### Node Details

#### Telegram: Receipt Photo
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Trigger node that starts the workflow when the Telegram bot receives a message.
- **Configuration choices:**
  - Listens to `message` updates.
  - `Download` is enabled in additional fields, so the incoming media is downloaded and passed as binary data.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input; this is an entry point.
  - Outputs to **Extract from File**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Invalid or missing Telegram bot credentials.
  - User sends a message without a photo or unsupported media type.
  - Telegram API/network issues.
  - If binary download fails, downstream file extraction will fail.
- **Sub-workflow reference:** None.

#### Extract from File
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Converts binary file content into a standard property.
- **Configuration choices:**
  - Operation: `binaryToPropery`
  - Binary property name: `data` via expression
- **Key expressions or variables used:**
  - `=data`
- **Input and output connections:**
  - Input from **Telegram: Receipt Photo**
  - Output to **Edit Fields**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - The expected binary property may not exist if Telegram did not provide or download the image correctly.
  - Non-image file structures may produce unexpected results.
  - Large files may increase memory usage.
- **Sub-workflow reference:** None.

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates or overwrites a `data` field formatted as a Base64 PNG data URI.
- **Configuration choices:**
  - Assigns one string field:
    - `data = data:image/png;base64,{{ $json.data }}`
- **Key expressions or variables used:**
  - `=data:image/png;base64,{{ $json.data }}`
- **Input and output connections:**
  - Input from **Extract from File**
  - Output to **HTTP Request**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Assumes the source image can be treated as PNG. If the uploaded receipt is JPEG or another format, the MIME prefix may be inaccurate even if the Base64 payload is valid.
  - If `$json.data` is missing or malformed, easybits will likely reject the payload.
- **Sub-workflow reference:** None.

---

## 2.2 AI Extraction via easybits

### Overview
This block sends the prepared image to an easybits extraction pipeline. The pipeline is expected to return structured fields under `json.data`, specifically invoice number, currency, and amount.

### Nodes Involved
- HTTP Request

### Node Details

#### HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the easybits Extractor API with the image content.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`
  - Authentication: predefined credential type using `httpBearerAuth`
  - Sends JSON body
  - Body structure:
    - `files` is an array containing the data URI from the previous node
- **Key expressions or variables used:**
  - JSON body includes `{{ $json.data }}`
- **Input and output connections:**
  - Input from **Edit Fields**
  - Output to **Parse AI Data**
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Missing or invalid Bearer token.
  - Placeholder pipeline ID not replaced.
  - easybits pipeline not configured to extract the expected fields.
  - API validation errors if the `files` array is malformed.
  - Timeouts or rate limits from the easybits service.
  - If the response structure differs from expected, parsing will fail downstream.
- **Sub-workflow reference:** None.

---

## 2.3 Data Normalization

### Overview
This block standardizes the easybits response. It handles the case where `json.data` may arrive as either a stringified JSON object or a native object, then applies fallback values for missing fields.

### Nodes Involved
- Parse AI Data

### Node Details

#### Parse AI Data
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that parses and normalizes extracted receipt data.
- **Configuration choices:**
  - Reads `let data = $input.first().json.data`
  - If `data` is a string, parses it with `JSON.parse(data)`
  - Returns:
    - `invoice_number` defaulting to `"UNKNOWN-ID"`
    - `currency` defaulting to `"USD"`
    - `amount` defaulting to `0`
- **Key expressions or variables used:**
  - `$input.first().json.data`
- **Input and output connections:**
  - Input from **HTTP Request**
  - Output to **Get Exchange Rate (Primary)**
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - `json.data` missing entirely.
  - Invalid JSON string causing `JSON.parse()` to throw.
  - `amount` returned as a non-numeric string; no explicit number coercion is done here.
  - If `currency` contains unexpected formatting or lowercase/whitespace, downstream API URLs may fail.
- **Sub-workflow reference:** None.

---

## 2.4 Exchange Rate Retrieval with Fallback

### Overview
This block fetches exchange rates based on the extracted invoice currency. It first queries a jsDelivr-hosted currency API, then checks whether the result indicates an error; if so, it calls a Cloudflare-hosted fallback endpoint.

### Nodes Involved
- Get Exchange Rate (Primary)
- Did Primary Fail?
- Fallback API (Cloudflare)

### Node Details

#### Get Exchange Rate (Primary)
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Requests exchange-rate data for the detected base currency.
- **Configuration choices:**
  - Dynamic URL:
    - `https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@latest/v1/currencies/{{ $json.currency.toLowerCase() }}.json`
- **Key expressions or variables used:**
  - `{{ $json.currency.toLowerCase() }}`
- **Input and output connections:**
  - Input from **Parse AI Data**
  - Output to **Did Primary Fail?**
- **Version-specific requirements:** Type version `4.1`.
- **Edge cases or potential failure types:**
  - If `currency` is null, malformed, or includes spaces/symbols, the URL may be invalid.
  - The external API may be unavailable, rate-limited, or change its response shape.
  - This workflow assumes an error indication may exist in `json.error`, but actual HTTP failures depend on node error behavior and settings.
- **Sub-workflow reference:** None.

#### Did Primary Fail?
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes execution depending on whether the primary response appears to contain an error.
- **Configuration choices:**
  - String condition:
    - `value1 = {{ $json.error }}`
    - Operation: `isNotEmpty`
  - If true: go to fallback API
  - If false: proceed directly to EUR calculation
- **Key expressions or variables used:**
  - `={{ $json.error }}`
- **Input and output connections:**
  - Input from **Get Exchange Rate (Primary)**
  - Output 1 (true) to **Fallback API (Cloudflare)**
  - Output 2 (false) to **Calculate EUR**
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If the primary HTTP node throws an execution error instead of returning JSON with `error`, this IF node may never run unless the HTTP node is configured to continue on fail.
  - A structurally invalid but non-error response may bypass fallback and fail later in conversion.
- **Sub-workflow reference:** None.

#### Fallback API (Cloudflare)
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Secondary exchange-rate request node used if the primary source indicates failure.
- **Configuration choices:**
  - Dynamic URL based on normalized currency from **Parse AI Data**:
    - `https://latest.currency-api.pages.dev/v1/currencies/{{ $node["Parse AI Data"].json.currency.toLowerCase() }}.json`
- **Key expressions or variables used:**
  - `{{ $node["Parse AI Data"].json.currency.toLowerCase() }}`
- **Input and output connections:**
  - Input from **Did Primary Fail?**
  - Output to **Calculate EUR**
- **Version-specific requirements:** Type version `4.1`.
- **Edge cases or potential failure types:**
  - Same currency-format risks as the primary API.
  - If the fallback also fails or returns a different structure, EUR conversion will fail.
  - Dependency on cross-node reference to **Parse AI Data** means renaming that node would break the expression unless updated.
- **Sub-workflow reference:** None.

---

## 2.5 EUR Conversion and Spreadsheet Logging

### Overview
This block extracts the EUR conversion rate from whichever exchange-rate API succeeded, computes the final EUR amount, and appends a row to the target Google Sheet.

### Nodes Involved
- Calculate EUR
- Append row in sheet

### Node Details

#### Calculate EUR
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that converts the invoice amount into EUR and shapes the final record for storage.
- **Configuration choices:**
  - Reads normalized invoice data from **Parse AI Data**
  - Reads exchange-rate JSON from the current input item
  - Uses:
    - `baseCurrency = invoice.currency.toLowerCase()`
    - `eurRate = rateData[baseCurrency].eur`
    - `amountInEur = invoice.amount * eurRate`
  - Returns:
    - `invoice_no`
    - `original_amount`
    - `original_currency`
    - `exchange_rate`
    - `amount_eur` rounded with `toFixed(2)`
- **Key expressions or variables used:**
  - `$node["Parse AI Data"].json`
  - `$input.all()[0].json`
- **Input and output connections:**
  - Input from **Did Primary Fail?** false branch or **Fallback API (Cloudflare)**
  - Output to **Append row in sheet**
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If the rate response does not contain `rateData[baseCurrency].eur`, the node will throw.
  - If `amount` is not numeric, multiplication may produce `NaN`.
  - `toFixed(2)` returns a string, not a numeric value.
  - Cross-node reference requires the node name **Parse AI Data** to remain stable.
- **Sub-workflow reference:** None.

#### Append row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the transformed expense record to a specific Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet: `gid=0` / `Sheet1`
  - Mapping mode: define below
  - Column mapping:
    - `Invoice Number` ← `{{$json.invoice_no}}`
    - `Original Amount` ← `{{$json.original_amount}}`
    - `Currency` ← `{{$json.original_currency}}`
    - `Exchange Rate` ← `{{$json.exchange_rate}}`
    - `Final Amount (EUR)` ← `{{$json.amount_eur}}`
- **Key expressions or variables used:**
  - `={{ $json.original_currency }}`
  - `={{ $json.exchange_rate }}`
  - `={{ $json.invoice_no }}`
  - `={{ $json.original_amount }}`
  - `={{ $json.amount_eur }}`
- **Input and output connections:**
  - Input from **Calculate EUR**
  - No downstream output.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Missing or invalid Google Sheets OAuth2 credentials.
  - Placeholder spreadsheet ID not replaced.
  - Sheet name or gid mismatch.
  - Expected columns not existing in the sheet.
  - Sticky note text mentions “Vendor Name” and “Overall Due”, but actual mapping uses invoice/currency/exchange columns; this mismatch can confuse setup.
- **Sub-workflow reference:** None.

---

## 2.6 Embedded Documentation / Visual Guidance

### Overview
These nodes do not affect execution. They provide setup notes, architecture explanations, and external resource links directly on the n8n canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for the Telegram trigger area.
- **Configuration choices:** Explains that the workflow starts from Telegram and requires photo download enabled.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes easybits extraction behavior and expected fields in `json.data`.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes normalization and fallback defaults.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the primary/fallback exchange-rate strategy.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents EUR conversion logic.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Google Sheets logging, but partially describes column mappings inconsistently with the actual node configuration.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime, but note the documentation mismatch.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Large general note describing workflow purpose and setup steps for easybits, Telegram, and Google Sheets.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram: Receipt Photo | Telegram Trigger | Entry point; receives Telegram messages and downloads receipt images |  | Extract from File | ## 📥 Entry Point<br>Listens for incoming Telegram messages. The user is expected to send a photo of a receipt or invoice. Make sure "Download" is enabled so the image binary is passed forward.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets.<br><br>**Flow overview:**<br>1. User sends a receipt photo via Telegram<br>2. easybits Extractor reads the document and returns structured data<br>3. The data is normalised and cleaned<br>4. The exchange rate is fetched (with fallback if needed)<br>5. The amount is converted to EUR<br>6. The result is appended to Google Sheets<br><br>### 3. Connect Your Telegram Bot<br>1. Open the **Telegram: Receipt Photo** node.<br>2. Connect your Telegram Bot credentials (Bot Token from [@BotFather](https://t.me/BotFather)).<br>3. Make sure **"Download"** is enabled under Additional Fields so the image binary is forwarded correctly.<br><br>### 5. Activate the Workflow<br>1. Click the **"Active"** toggle in the top-right corner of n8n to enable the workflow.<br>2. Send a receipt photo to your Telegram bot to test it end to end.<br>3. Check your Google Sheet – a new row with the invoice reference and EUR amount should appear. |
| Extract from File | Extract From File | Converts downloaded binary into a property for reuse | Telegram: Receipt Photo | Edit Fields | ## 🤖 easybits' Data Extraction<br>Sends the receipt image to the easybits pipeline for data extraction. Returns structured data under `json.data` containing `invoice_number`, `currency`, and `amount`.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| Edit Fields | Set | Rebuilds image content as a Base64 data URI | Extract from File | HTTP Request | ## 🤖 easybits' Data Extraction<br>Sends the receipt image to the easybits pipeline for data extraction. Returns structured data under `json.data` containing `invoice_number`, `currency`, and `amount`.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| HTTP Request | HTTP Request | Sends receipt image to easybits extraction API | Edit Fields | Parse AI Data | ## 🤖 easybits' Data Extraction<br>Sends the receipt image to the easybits pipeline for data extraction. Returns structured data under `json.data` containing `invoice_number`, `currency`, and `amount`.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets.<br><br>## Step-by-Step Setup Guide<br>### 1. Set Up Your easybits Extractor Pipeline<br>Go to [extractor.easybits.tech](https://extractor.easybits.tech) and create a pipeline with fields `invoice_number`, `currency`, and `amount`.<br><br>### 2. Connect the easybits Node in n8n<br>Use API URL `https://extractor.easybits.tech/api/pipelines/[YOUR_PIPELINE_ID]` and configure Bearer Auth with your API key. |
| Parse AI Data | Code | Parses and normalizes extracted invoice fields | HTTP Request | Get Exchange Rate (Primary) | ## 🧹 Data Normalisation<br>Cleans and standardises the extracted fields. Provides fallback defaults in case the AI missed a field (e.g. unknown invoice number, USD as default currency).<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| Get Exchange Rate (Primary) | HTTP Request | Fetches exchange rates from the primary API | Parse AI Data | Did Primary Fail? | ## 🌐 Exchange Rate Fetch (with Fallback)<br>Tries the primary jsDelivr-hosted currency API first. If it fails, the error output automatically reroutes to the Cloudflare fallback API. Both return the same response structure so `Calculate EUR` works identically either way.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| Did Primary Fail? | If | Tests whether the primary rate response contains an error | Get Exchange Rate (Primary) | Fallback API (Cloudflare), Calculate EUR | ## 🌐 Exchange Rate Fetch (with Fallback)<br>Tries the primary jsDelivr-hosted currency API first. If it fails, the error output automatically reroutes to the Cloudflare fallback API. Both return the same response structure so `Calculate EUR` works identically either way.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| Fallback API (Cloudflare) | HTTP Request | Fetches exchange rates from the fallback API | Did Primary Fail? | Calculate EUR | ## 🌐 Exchange Rate Fetch (with Fallback)<br>Tries the primary jsDelivr-hosted currency API first. If it fails, the error output automatically reroutes to the Cloudflare fallback API. Both return the same response structure so `Calculate EUR` works identically either way.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| Calculate EUR | Code | Computes converted EUR amount and final output payload | Did Primary Fail?, Fallback API (Cloudflare) | Append row in sheet | ## 🧮 Currency Conversion<br>Reads the exchange rate from whichever API succeeded and multiplies it by the original invoice amount to produce the final EUR value. Result is rounded to 2 decimal places.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets. |
| Append row in sheet | Google Sheets | Appends the converted expense record to a spreadsheet | Calculate EUR |  | ## 📊 Google Sheets Logging<br>Appends one row per invoice to the Master Finance File. Maps `invoice_no` → Vendor Name and `amount_eur` → Overall Due. Make sure the sheet columns exist before running.<br><br># 💱 Currency Converter Assistant<br>## How It Works<br>This workflow automates multi-currency expense tracking via Telegram. Send a receipt photo to your bot, and it automatically extracts the invoice details, converts the amount to EUR using a live exchange rate, and logs everything straight into Google Sheets.<br><br>### 4. Connect Google Sheets<br>1. Open the **Append row in sheet** node.<br>2. Connect your Google Sheets account via OAuth2.<br>3. Select your target spreadsheet and sheet.<br>4. Make sure your sheet has at least these two columns: **Vendor Name** and **Overall Due**. |
| Sticky Note | Sticky Note | Canvas documentation for entry point |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for easybits extraction block |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for normalization block |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for exchange-rate block |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for conversion block |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation for Google Sheets logging block |  |  |  |
| Sticky Note6 | Sticky Note | Global canvas documentation and setup notes |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `Receipt-to-Sheet: Multi-Currency Expense Tracker with easybits & Telegram`.

2. **Add a Telegram Trigger node** named **Telegram: Receipt Photo**.
   - Node type: `Telegram Trigger`
   - Configure:
     - Updates: `message`
     - Additional Fields → enable `Download`
   - Credentials:
     - Create/select Telegram bot credentials using your Bot Token from [@BotFather](https://t.me/BotFather).
   - This is the workflow entry point.

3. **Add an Extract From File node** named **Extract from File**.
   - Node type: `Extract From File`
   - Operation: `Binary to Property`
   - Binary Property Name: `data`
   - Connect:
     - **Telegram: Receipt Photo** → **Extract from File**

4. **Add a Set node** named **Edit Fields**.
   - Node type: `Set`
   - Create one assignment:
     - Field name: `data`
     - Type: `String`
     - Value:
       ```text
       data:image/png;base64,{{ $json.data }}
       ```
   - Connect:
     - **Extract from File** → **Edit Fields**

5. **Create your easybits extraction pipeline** before configuring the API call.
   - Go to [https://extractor.easybits.tech](https://extractor.easybits.tech)
   - Create a pipeline for receipt/invoice extraction.
   - Define at least these fields:
     - `invoice_number` as string
     - `currency` as string
     - `amount` as number
   - Test the pipeline with a sample receipt.
   - Copy:
     - Pipeline API URL
     - API key

6. **Add an HTTP Request node** named **HTTP Request** for easybits.
   - Node type: `HTTP Request`
   - Method: `POST`
   - URL:
     ```text
     https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID
     ```
     Replace `YOUR_PIPELINE_ID` with your actual pipeline ID.
   - Authentication:
     - Use predefined credential type
     - Credential type: `HTTP Bearer Auth`
     - Create credentials with your easybits API key as the Bearer token
   - Body settings:
     - Send Body: enabled
     - Specify Body: `JSON`
     - JSON Body:
       ```json
       {
         "files": [
           "{{ $json.data }}"
         ]
       }
       ```
   - Connect:
     - **Edit Fields** → **HTTP Request**

7. **Add a Code node** named **Parse AI Data**.
   - Node type: `Code`
   - JavaScript:
     ```javascript
     // Safely capture the extracted data from easybits Extractor
     let data = $input.first().json.data;

     // Sometimes AI returns it as a string, sometimes as an object
     if (typeof data === 'string') {
       data = JSON.parse(data);
     }

     return {
       invoice_number: data.invoice_number || "UNKNOWN-ID",
       currency: data.currency || "USD",
       amount: data.amount || 0
     };
     ```
   - Connect:
     - **HTTP Request** → **Parse AI Data**

8. **Add a primary HTTP Request node** named **Get Exchange Rate (Primary)**.
   - Node type: `HTTP Request`
   - Method: `GET`
   - URL:
     ```text
     https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@latest/v1/currencies/{{ $json.currency.toLowerCase() }}.json
     ```
   - No authentication required.
   - Connect:
     - **Parse AI Data** → **Get Exchange Rate (Primary)**

9. **Add an If node** named **Did Primary Fail?**.
   - Node type: `If`
   - Condition:
     - Type: String
     - Value 1:
       ```text
       {{ $json.error }}
       ```
     - Operation: `is not empty`
   - Connect:
     - **Get Exchange Rate (Primary)** → **Did Primary Fail?**

10. **Add a fallback HTTP Request node** named **Fallback API (Cloudflare)**.
    - Node type: `HTTP Request`
    - Method: `GET`
    - URL:
      ```text
      https://latest.currency-api.pages.dev/v1/currencies/{{ $node["Parse AI Data"].json.currency.toLowerCase() }}.json
      ```
    - No authentication required.
    - Connect:
      - **Did Primary Fail?** true branch → **Fallback API (Cloudflare)**

11. **Add a Code node** named **Calculate EUR**.
    - Node type: `Code`
    - JavaScript:
      ```javascript
      const invoice = $node["Parse AI Data"].json;
      const baseCurrency = invoice.currency.toLowerCase();

      // Capture the API response from whichever node succeeded
      const rateData = $input.all()[0].json;

      // Extract the EUR rate dynamically
      const eurRate = rateData[baseCurrency].eur;
      const amountInEur = invoice.amount * eurRate;

      return {
        invoice_no: invoice.invoice_number,
        original_amount: invoice.amount,
        original_currency: invoice.currency.toUpperCase(),
        exchange_rate: eurRate,
        amount_eur: amountInEur.toFixed(2)
      };
      ```
    - Connect:
      - **Did Primary Fail?** false branch → **Calculate EUR**
      - **Fallback API (Cloudflare)** → **Calculate EUR**

12. **Prepare the target Google Sheet**.
    - Create a spreadsheet, for example `Master Finance File`.
    - In the destination sheet, create these columns exactly:
      - `Invoice Number`
      - `Original Amount`
      - `Currency`
      - `Exchange Rate`
      - `Final Amount (EUR)`
    - Important:
      - The sticky note text mentions `Vendor Name` and `Overall Due`, but the actual node mapping uses the five columns above. Use the actual mapping.

13. **Add a Google Sheets node** named **Append row in sheet**.
    - Node type: `Google Sheets`
    - Operation: `Append`
    - Authenticate with Google Sheets OAuth2 credentials.
    - Select your target spreadsheet.
    - Select the destination sheet.
    - Use mapping mode: `Define Below`
    - Map columns:
      - `Invoice Number` → `{{ $json.invoice_no }}`
      - `Original Amount` → `{{ $json.original_amount }}`
      - `Currency` → `{{ $json.original_currency }}`
      - `Exchange Rate` → `{{ $json.exchange_rate }}`
      - `Final Amount (EUR)` → `{{ $json.amount_eur }}`
    - Connect:
      - **Calculate EUR** → **Append row in sheet**

14. **Optionally add sticky notes** to mirror the original canvas documentation.
    - Add notes for:
      - Telegram entry point
      - easybits extraction
      - data normalization
      - exchange-rate fallback
      - EUR conversion
      - Google Sheets logging
      - overall setup notes

15. **Test the workflow manually**.
    - Send a photo to the Telegram bot.
    - Confirm that:
      - binary data is present after the Telegram trigger
      - easybits returns `json.data`
      - Parse AI Data outputs valid `invoice_number`, `currency`, and `amount`
      - one exchange-rate API returns valid rates
      - Calculate EUR returns numeric conversion results
      - the Google Sheets node appends a row successfully

16. **Activate the workflow** once end-to-end testing succeeds.

## Credential Configuration Summary

### Telegram
- Credential type: Telegram API / bot token
- Source: [@BotFather](https://t.me/BotFather)

### easybits
- Credential type: HTTP Bearer Auth
- Bearer token: the API key provided by your easybits pipeline

### Google Sheets
- Credential type: Google Sheets OAuth2
- Must have access to the target spreadsheet

## Input/Output Expectations

### Expected Telegram input
- A Telegram message containing a receipt or invoice image

### Expected easybits output
- Response should contain `json.data`
- `json.data` should be either:
  - an object with `invoice_number`, `currency`, `amount`
  - or a JSON string representing the same object

### Expected exchange-rate output
- JSON structure like:
  ```json
  {
    "usd": {
      "eur": 0.92
    }
  }
  ```
  where the top-level key matches the lowercase base currency

### Final Google Sheets output
- One appended row per processed receipt

## Important Rebuild Considerations

1. **The current fallback logic is response-based, not exception-based.**
   - If the primary HTTP node throws a hard execution error, the If node may not run.
   - For production hardening, consider enabling error handling or using an error branch strategy.

2. **The image MIME type is hardcoded to PNG.**
   - If Telegram delivers JPEGs, consider dynamically setting the MIME type.

3. **Currency validation is minimal.**
   - It may be wise to uppercase/trim the extracted code and validate against ISO currency codes before calling APIs.

4. **Amount type is not strongly normalized.**
   - If easybits returns `"149.99"` as a string, multiplication may still work, but locale-formatted values like `"149,99"` will not.

5. **Node-name references matter.**
   - Expressions in **Fallback API (Cloudflare)** and **Calculate EUR** explicitly reference `Parse AI Data`.
   - If you rename that node, update all dependent expressions.

6. **No sub-workflows are used.**
   - This workflow has a single entry point and no Execute Workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| easybits extractor service for building the document extraction pipeline | [https://extractor.easybits.tech](https://extractor.easybits.tech) |
| Telegram bot token setup via BotFather | [https://t.me/BotFather](https://t.me/BotFather) |
| The embedded documentation describes the workflow as a “Currency Converter Assistant” | Internal canvas note |
| The sticky note for Google Sheets contains a mapping inconsistency: it mentions `invoice_no` → Vendor Name and `amount_eur` → Overall Due, while the actual node maps to `Invoice Number`, `Original Amount`, `Currency`, `Exchange Rate`, and `Final Amount (EUR)` | Important implementation note |
| The workflow is inactive in the provided export (`active: false`) | Deployment state |
| The workflow uses one entry point only: Telegram Trigger | Architecture note |