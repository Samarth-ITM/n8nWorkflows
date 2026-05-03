Validate JSON and CSV import data via webhook with configurable rules

https://n8nworkflows.xyz/workflows/validate-json-and-csv-import-data-via-webhook-with-configurable-rules-13999


# Validate JSON and CSV import data via webhook with configurable rules

## 1. Workflow Overview

This workflow exposes a reusable HTTP validation endpoint in n8n. It accepts a `POST` request containing a `data` array and optional `rules`, validates each row against configurable field-level rules, and returns a structured JSON report summarizing valid and invalid records.

Typical use cases include validating incoming import batches before inserting them into CRM, ERP, spreadsheet, or database systems. The workflow is designed as a generic pre-import validation service rather than as a file parser itself. Although the title mentions CSV and JSON import data, the webhook expects JSON in the request body; CSV data would need to be converted into JSON records before or before calling this endpoint.

### 1.1 Input Reception
The workflow starts with a webhook endpoint that receives the request body. The expected payload contains:
- `data`: an array of row objects
- `rules`: an optional object defining validation rules per field

### 1.2 Default Rule Provisioning
A Set node defines fallback validation rules. These defaults are used when the incoming request does not provide its own `rules` object.

### 1.3 Row and Field Validation
A Code node performs all validation logic:
- checks the request structure
- merges request rules with fallback behavior
- validates each field in each row
- collects detailed errors
- produces a final report

### 1.4 API Response
A Respond to Webhook node returns the validation report as JSON with an explicit `Content-Type: application/json` header.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception

**Overview:**  
This block receives the HTTP request and serves as the workflow entry point. It is configured to defer the HTTP response until a later response node returns the validation report.

**Nodes Involved:**  
- Receive Data

#### Node: Receive Data
- **Type and technical role:** `n8n-nodes-base.webhook`  
  HTTP entry point for external callers.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `validate-data`
  - Response mode: `responseNode`
  This means the webhook waits for a downstream Respond to Webhook node instead of auto-responding immediately.
- **Key expressions or variables used:**  
  No expressions inside the node itself. Its output is later accessed in the Code node via:
  - `$('Receive Data').first().json.body`
- **Input and output connections:**
  - Input: none, this is an entry point
  - Output: connected to `Set Default Rules`
- **Version-specific requirements:**  
  Uses webhook node version `2.1`. The `responseNode` mode requires a compatible downstream `Respond to Webhook` node.
- **Edge cases or potential failure types:**
  - Wrong HTTP method will not match the webhook
  - If callers send malformed JSON, n8n may reject the request before workflow logic runs
  - If the workflow is inactive, production webhook calls will fail
  - If no Respond to Webhook node is reached, the request may hang or fail
- **Sub-workflow reference:**  
  None

---

### Block 2 — Default Rule Provisioning

**Overview:**  
This block provides a built-in fallback ruleset. It allows the validation service to work even when clients omit `rules` from the request body.

**Nodes Involved:**  
- Set Default Rules

#### Node: Set Default Rules
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a JSON object containing default validation rules.
- **Configuration choices:**
  - Mode: `raw`
  - JSON output contains a `rules` object with field-specific rules for:
    - `name`
    - `email`
    - `age`
    - `status`
    - `website`
    - `joinDate`
- **Key expressions or variables used:**  
  No expressions. Static JSON is emitted.
- **Configured default rules:**
  - `name`: required string, min length 2, max length 100
  - `email`: required email
  - `age`: optional number, min 0, max 150
  - `status`: required string, enum of `active`, `inactive`, `pending`
  - `website`: optional URL
  - `joinDate`: optional date with format `YYYY-MM-DD`
- **Input and output connections:**
  - Input: `Receive Data`
  - Output: `Validate Data`
- **Version-specific requirements:**  
  Uses Set node version `3.4` with raw JSON output support.
- **Edge cases or potential failure types:**
  - Invalid JSON in raw mode would break execution
  - Misconfigured default rules can cause false positives or false negatives
  - There is no schema validation on the rules object itself, so malformed rule definitions are passed directly to the Code node
- **Sub-workflow reference:**  
  None

---

### Block 3 — Row and Field Validation

**Overview:**  
This is the core logic block. It reads the webhook payload and fallback rules, validates every configured field for every row, gathers all failures, and generates a final summary.

**Nodes Involved:**  
- Validate Data

#### Node: Validate Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes custom JavaScript to implement the full validation engine.
- **Configuration choices:**
  - Reads request body from the webhook node
  - Reads default rules from the Set node
  - Uses request `rules` if provided; otherwise uses `defaults.rules`
  - Returns a single JSON item containing:
    - `valid`
    - `summary`
    - `errors`
    - and in some failure branches, a top-level `error`
- **Key expressions or variables used:**
  - `const input = $('Receive Data').first().json.body;`
  - `const defaults = $('Set Default Rules').first().json;`
  - `const data = input.data;`
  - `const rules = input.rules || defaults.rules || {};`
- **Validation behavior implemented:**
  - Input validation:
    - `data` must exist and be an array
    - `rules` must exist either in the request or in defaults
  - Presence check:
    - `required`
  - Type checks:
    - `string`
    - `number`
    - `email`
    - `url`
    - `boolean`
    - `date`
  - Date format support:
    - `YYYY-MM-DD`
    - `DD.MM.YYYY`
    - `MM/DD/YYYY`
    - otherwise falls back to `Date.parse`
  - Numeric checks:
    - `min`
    - `max`
  - String length checks:
    - `minLength`
    - `maxLength`
  - Pattern check:
    - `regex`
  - Enumerated values:
    - `enum`
- **Input and output connections:**
  - Input: `Set Default Rules`
  - Output: `Respond with Report`
- **Version-specific requirements:**  
  Uses Code node version `2`. JavaScript execution must be supported in the n8n instance.
- **Edge cases or potential failure types:**
  - If `input` is missing or `body` is undefined, expressions could fail depending on webhook payload shape
  - Request `rules` entirely replace defaults; they do not merge field-by-field
  - Rows are assumed to be objects; there is no explicit check that each array entry is a plain object
  - `string` type validation requires actual JavaScript strings, so numeric values like `123` fail even if convertible
  - `number` accepts numeric strings such as `"42"`
  - `boolean` accepts only string-like values in `true`, `false`, `0`, `1`, `yes`, `no`; actual booleans stringify correctly, but other conventions are rejected
  - URL validation is intentionally simple and may reject some valid URLs or accept some loosely formed ones
  - Date validation for formatted dates only checks regex shape, not true calendar validity, so invalid dates like `2024-99-99` may pass format validation
  - `regex` rules can throw if the supplied pattern is invalid because `new RegExp(fr.regex)` is not wrapped in try/catch
  - `enum` uses strict inclusion with original value types, so `"1"` and `1` are different
  - Multiple errors can be reported for the same field in the same row
  - Unknown `type` values are silently ignored because the switch has no default rejection branch
- **Sub-workflow reference:**  
  None

---

### Block 4 — API Response

**Overview:**  
This block sends the validation outcome back to the caller. It returns the Code node’s JSON as the HTTP response body.

**Nodes Involved:**  
- Respond with Report

#### Node: Respond with Report
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Finalizes the webhook request with a JSON response.
- **Configuration choices:**
  - Respond with: `json`
  - Response body: `={{ $json }}`
  - Custom response header:
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `={{ $json }}`
- **Input and output connections:**
  - Input: `Validate Data`
  - Output: none
- **Version-specific requirements:**  
  Uses Respond to Webhook node version `1.5`. Requires the upstream webhook to be configured in `responseNode` mode.
- **Edge cases or potential failure types:**
  - If upstream nodes fail, this node is never reached and the webhook request returns an error instead of the intended report
  - Manual header configuration duplicates behavior normally implied by JSON response mode, but is harmless
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Data | Webhook | Receives POST requests containing `data` and optional `rules` |  | Set Default Rules | ## Validate CSV/JSON Import Data with Configurable Rules<br><br>This workflow provides a **reusable data validation API endpoint** for your import pipelines. Send any JSON array of records along with validation rules, and get back a detailed report showing which rows passed and which failed, with specific error messages per field.<br><br>### Who is this for?<br>Operations teams, data engineers, or anyone importing data into ERP, CRM, databases, or spreadsheets who needs to catch errors **before** they enter the system.<br><br>### How it works<br>1. Send a POST request with `data` (array of records) and `rules` (validation config)<br>2. The workflow validates every field in every row against your rules<br>3. Returns a structured JSON report: total/valid/invalid rows + detailed errors<br><br>### Supported validation rules<br>`required`, `type` (string, number, email, date, url, boolean), `min`, `max`, `minLength`, `maxLength`, `regex`, `enum`, `dateFormat`<br><br>### Setup<br>1. Activate the workflow<br>2. Send a POST request to the webhook URL<br>3. Optionally configure default rules in the **Set Default Rules** node<br><br>**Author:** Florian Eiche, [eiche-digital.de](https://eiche-digital.de)<br><br>### Step 1: Receive Data<br>POST request with JSON body:<br>```json<br>{<br>  "data": [{...}, {...}],<br>  "rules": {<br>    "fieldName": {<br>      "required": true,<br>      "type": "email"<br>    }<br>  }<br>}<br>```<br>Rules in the request override the defaults. |
| Set Default Rules | Set | Defines fallback validation rules | Receive Data | Validate Data | ## Validate CSV/JSON Import Data with Configurable Rules<br><br>This workflow provides a **reusable data validation API endpoint** for your import pipelines. Send any JSON array of records along with validation rules, and get back a detailed report showing which rows passed and which failed, with specific error messages per field.<br><br>### Who is this for?<br>Operations teams, data engineers, or anyone importing data into ERP, CRM, databases, or spreadsheets who needs to catch errors **before** they enter the system.<br><br>### How it works<br>1. Send a POST request with `data` (array of records) and `rules` (validation config)<br>2. The workflow validates every field in every row against your rules<br>3. Returns a structured JSON report: total/valid/invalid rows + detailed errors<br><br>### Supported validation rules<br>`required`, `type` (string, number, email, date, url, boolean), `min`, `max`, `minLength`, `maxLength`, `regex`, `enum`, `dateFormat`<br><br>### Setup<br>1. Activate the workflow<br>2. Send a POST request to the webhook URL<br>3. Optionally configure default rules in the **Set Default Rules** node<br><br>**Author:** Florian Eiche, [eiche-digital.de](https://eiche-digital.de)<br><br>### Step 2: Default Rules<br>Configure fallback validation rules here. These are used when the request body does not include a `rules` object.<br><br>Edit the JSON in the **Set Default Rules** node to match your data structure. |
| Validate Data | Code | Validates all rows against configured rules and builds the report | Set Default Rules | Respond with Report | ## Validate CSV/JSON Import Data with Configurable Rules<br><br>This workflow provides a **reusable data validation API endpoint** for your import pipelines. Send any JSON array of records along with validation rules, and get back a detailed report showing which rows passed and which failed, with specific error messages per field.<br><br>### Who is this for?<br>Operations teams, data engineers, or anyone importing data into ERP, CRM, databases, or spreadsheets who needs to catch errors **before** they enter the system.<br><br>### How it works<br>1. Send a POST request with `data` (array of records) and `rules` (validation config)<br>2. The workflow validates every field in every row against your rules<br>3. Returns a structured JSON report: total/valid/invalid rows + detailed errors<br><br>### Supported validation rules<br>`required`, `type` (string, number, email, date, url, boolean), `min`, `max`, `minLength`, `maxLength`, `regex`, `enum`, `dateFormat`<br><br>### Setup<br>1. Activate the workflow<br>2. Send a POST request to the webhook URL<br>3. Optionally configure default rules in the **Set Default Rules** node<br><br>**Author:** Florian Eiche, [eiche-digital.de](https://eiche-digital.de)<br><br>### Step 3: Validate<br>The Code node checks every row against the rules and collects all errors.<br><br>Supported checks:<br>- `required`<br>- `type`: string, number, email, date, url, boolean<br>- `min` / `max` (numbers)<br>- `minLength` / `maxLength`<br>- `regex` (custom pattern)<br>- `enum` (allowed values)<br>- `dateFormat` (YYYY-MM-DD, DD.MM.YYYY, MM/DD/YYYY) |
| Respond with Report | Respond to Webhook | Returns the JSON validation report to the HTTP caller | Validate Data |  | ## Validate CSV/JSON Import Data with Configurable Rules<br><br>This workflow provides a **reusable data validation API endpoint** for your import pipelines. Send any JSON array of records along with validation rules, and get back a detailed report showing which rows passed and which failed, with specific error messages per field.<br><br>### Who is this for?<br>Operations teams, data engineers, or anyone importing data into ERP, CRM, databases, or spreadsheets who needs to catch errors **before** they enter the system.<br><br>### How it works<br>1. Send a POST request with `data` (array of records) and `rules` (validation config)<br>2. The workflow validates every field in every row against your rules<br>3. Returns a structured JSON report: total/valid/invalid rows + detailed errors<br><br>### Supported validation rules<br>`required`, `type` (string, number, email, date, url, boolean), `min`, `max`, `minLength`, `maxLength`, `regex`, `enum`, `dateFormat`<br><br>### Setup<br>1. Activate the workflow<br>2. Send a POST request to the webhook URL<br>3. Optionally configure default rules in the **Set Default Rules** node<br><br>**Author:** Florian Eiche, [eiche-digital.de](https://eiche-digital.de)<br><br>### Step 4: Response<br>Returns a JSON report:<br>```json<br>{<br>  "valid": false,<br>  "summary": {<br>    "totalRows": 3,<br>    "validRows": 1,<br>    "invalidRows": 2,<br>    "totalErrors": 4<br>  },<br>  "errors": [<br>    {<br>      "row": 2,<br>      "field": "email",<br>      "value": "invalid",<br>      "rule": "type:email",<br>      "message": "..."<br>    }<br>  ]<br>}<br>``` |
| Sticky Note | Sticky Note | Canvas documentation for overall workflow purpose and setup |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for request payload format |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for fallback rule configuration |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for validation capabilities |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for response format |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   Give it a name such as:  
   `Validate CSV and JSON import data with configurable rules via webhook`

2. **Add a Webhook node**
   - Node type: **Webhook**
   - Name: `Receive Data`
   - Set **HTTP Method** to `POST`
   - Set **Path** to `validate-data`
   - Set **Response Mode** to `Using Respond to Webhook Node` or equivalent `responseNode` mode
   - Leave other options at default unless your environment requires authentication, CORS, or custom response settings

3. **Define the expected request payload**
   The webhook should accept a JSON body like:
   ```json
   {
     "data": [
       {
         "name": "Alice",
         "email": "alice@example.com",
         "age": 30,
         "status": "active",
         "website": "https://example.com",
         "joinDate": "2024-01-15"
       }
     ],
     "rules": {
       "email": {
         "required": true,
         "type": "email"
       }
     }
   }
   ```
   Notes:
   - `data` is required and must be an array
   - `rules` is optional if default rules are configured downstream
   - If `rules` is present, it replaces the default rules entirely in this implementation

4. **Add a Set node**
   - Node type: **Set**
   - Name: `Set Default Rules`
   - Connect `Receive Data` → `Set Default Rules`
   - Set mode to **Raw**
   - Enable JSON output and paste a fallback rules object like this:
   ```json
   {
     "rules": {
       "name": {
         "required": true,
         "type": "string",
         "minLength": 2,
         "maxLength": 100
       },
       "email": {
         "required": true,
         "type": "email"
       },
       "age": {
         "required": false,
         "type": "number",
         "min": 0,
         "max": 150
       },
       "status": {
         "required": true,
         "type": "string",
         "enum": ["active", "inactive", "pending"]
       },
       "website": {
         "required": false,
         "type": "url"
       },
       "joinDate": {
         "required": false,
         "type": "date",
         "dateFormat": "YYYY-MM-DD"
       }
     }
   }
   ```
   Adjust these fields to match your target import schema.

5. **Add a Code node**
   - Node type: **Code**
   - Name: `Validate Data`
   - Connect `Set Default Rules` → `Validate Data`
   - Language: **JavaScript**
   - Paste the validation logic so it:
     - reads the webhook body from `Receive Data`
     - reads fallback rules from `Set Default Rules`
     - assigns `const data = input.data`
     - assigns `const rules = input.rules || defaults.rules || {}`
     - validates data structure
     - loops through each row and each field rule
     - records validation errors
     - returns a single JSON item containing summary and errors

6. **Implement the Code node logic**
   Your code should include these major parts:

   1. **Read upstream data**
      - Request body from the webhook node
      - Default rules from the Set node

   2. **Choose rules**
      - Use request rules if provided
      - Otherwise use default rules

   3. **Validate top-level request**
      - If `data` is missing or not an array, return:
        - `valid: false`
        - a top-level `error` string
        - `summary`
        - empty `errors` array
      - If no rules exist, return an equivalent failure object

   4. **Create helper functions**
      - presence check
      - email test
      - number test
      - URL test
      - boolean-like test
      - date-format test

   5. **Loop through records**
      - For each row:
        - evaluate every field present in `rules`
        - push error objects with:
          - `row`
          - `field`
          - `value`
          - `rule`
          - `message`

   6. **Return final output**
      - `valid`: `errors.length === 0`
      - `summary`:
        - `totalRows`
        - `validRows`
        - `invalidRows`
        - `totalErrors`
      - `errors`: array of detailed validation failures

7. **Use the following validation semantics in the Code node**
   Support these rule keys:
   - `required`
   - `type`
   - `min`
   - `max`
   - `minLength`
   - `maxLength`
   - `regex`
   - `enum`
   - `dateFormat`

   Support these `type` values:
   - `string`
   - `number`
   - `email`
   - `date`
   - `url`
   - `boolean`

8. **Recommended behavior for helper checks**
   Recreate the logic approximately as follows:
   - A value is considered present if it is not `null`, not `undefined`, and not an empty string
   - Email validation uses a simple regex
   - Number validation accepts numeric strings
   - URL validation expects `http://` or `https://`
   - Boolean validation accepts:
     - `true`
     - `false`
     - `0`
     - `1`
     - `yes`
     - `no`
   - Date validation supports:
     - `YYYY-MM-DD`
     - `DD.MM.YYYY`
     - `MM/DD/YYYY`
     - fallback to JavaScript `Date.parse` when no format is specified

9. **Add a Respond to Webhook node**
   - Node type: **Respond to Webhook**
   - Name: `Respond with Report`
   - Connect `Validate Data` → `Respond with Report`
   - Set **Respond With** to `JSON`
   - Set **Response Body** to:
     - `={{ $json }}`
   - Add a response header:
     - `Content-Type` = `application/json`

10. **Verify connection order**
    The final chain should be:
    - `Receive Data` → `Set Default Rules` → `Validate Data` → `Respond with Report`

11. **Optionally add sticky notes for documentation**
    Add notes describing:
    - request structure
    - default rule editing
    - supported validation checks
    - output structure
    This is optional for execution but useful for maintainability.

12. **Activate the workflow**
    Activation is required if you want to call the production webhook URL.

13. **Test with a valid request**
    Example:
    ```json
    {
      "data": [
        {
          "name": "Alice",
          "email": "alice@example.com",
          "age": 30,
          "status": "active",
          "website": "https://example.com",
          "joinDate": "2024-01-15"
        }
      ]
    }
    ```
    Expected result:
    - `valid: true`
    - `validRows: 1`
    - `invalidRows: 0`
    - `errors: []`

14. **Test with an invalid request**
    Example:
    ```json
    {
      "data": [
        {
          "name": "A",
          "email": "invalid",
          "age": 200,
          "status": "unknown",
          "website": "not-a-url",
          "joinDate": "15/01/2024"
        }
      ]
    }
    ```
    Expected result:
    - `valid: false`
    - one invalid row
    - multiple field-level error entries

15. **Be aware of implementation constraints**
    - Request `rules` override defaults completely; they do not merge
    - The workflow validates JSON objects, not raw CSV files
    - If you need CSV support, add a prior parsing layer that converts CSV rows into JSON before calling or entering this workflow
    - Invalid regular expression patterns in `rules.regex` can break execution unless you add error handling
    - Date format checks are pattern-based, not full calendar validation
    - Unknown `type` values are not explicitly rejected in this implementation

16. **No credentials are required**
    - This workflow uses no external service nodes
    - No OAuth2, API keys, or database credentials are needed
    - If you want webhook protection, add authentication at the webhook or reverse-proxy level

17. **No sub-workflows are required**
    - This workflow is self-contained
    - It does not call other workflows
    - It does not expose callable sub-workflow interfaces beyond the webhook itself

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: Florian Eiche | Workflow annotation |
| eiche-digital.de | https://eiche-digital.de |
| The workflow title mentions CSV and JSON import data, but the actual endpoint validates JSON payloads. CSV must be transformed into JSON records before validation. | Implementation note |
| The webhook path is `validate-data`. | Endpoint configuration |
| Supported checks documented in the workflow: `required`, `type`, `min`, `max`, `minLength`, `maxLength`, `regex`, `enum`, `dateFormat`. | Validation capabilities |
| Response structure includes `valid`, `summary`, and `errors`; some early failure cases also include a top-level `error` field. | Output contract |