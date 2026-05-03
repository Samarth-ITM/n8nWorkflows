Transform and validate webhook records with configurable type conversion

https://n8nworkflows.xyz/workflows/transform-and-validate-webhook-records-with-configurable-type-conversion-14034


# Transform and validate webhook records with configurable type conversion

# 1. Workflow Overview

This workflow is a standalone data transformation and validation pipeline exposed through an HTTP webhook. It receives one or more input records, applies a configurable field-mapping scheme, converts values into target data types, validates required fields, and returns a structured JSON response containing transformed records plus any validation or conversion errors.

It is designed for teams that need to normalize inbound data before loading it into another system such as a CRM, ERP, database, or custom API. It does not rely on external services, credentials, or third-party APIs.

## 1.1 Input Reception

The workflow starts with a webhook that accepts `POST` requests. The payload can be either:
- an object with a `records` array, or
- a single object representing one record.

The workflow is configured to return its response through a dedicated Respond to Webhook node.

## 1.2 Mapping Configuration

A Set node stores the transformation rules as inline JSON. These rules define:
- source field names
- output field names
- target data types
- default values
- required-field behavior
- global conversion settings

This makes the workflow configurable without editing the transformation logic itself.

## 1.3 Record Transformation and Validation

A Code node performs the main logic:
- normalizes input into an array of records
- applies each field mapping
- trims strings if enabled
- converts empty strings to `null` if enabled
- applies defaults
- validates required fields
- converts values to `string`, `number`, `boolean`, or `date`
- optionally removes unmapped fields
- aggregates row-level errors

The result is a single JSON object summarizing total records, valid records, error count, transformed data, and all generated errors.

## 1.4 HTTP Response

A Respond to Webhook node returns the final JSON payload to the original caller with `Content-Type: application/json`.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block exposes the workflow as an HTTP endpoint. It accepts incoming data and passes execution into the mapping and transformation stages.

### Nodes Involved
- Receive Data

### Node Details

#### Receive Data
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that listens for incoming HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `transform-data`
  - Response mode: `responseNode`
- **Key expressions or variables used:**
  - No expressions in the node itself
  - Downstream code references this node with:
    - `$('Receive Data').first().json.body`
    - `$('Receive Data').first().json`
- **Input and output connections:**
  - Input: none
  - Output: `Configure Field Mapping`
- **Version-specific requirements:**
  - Uses node type version `2.1`
  - Because response mode is `responseNode`, the workflow must include a Respond to Webhook node to complete the HTTP request properly
- **Edge cases or potential failure types:**
  - If the workflow is inactive, the production webhook will not receive requests
  - If request body parsing does not produce the expected JSON structure, downstream transformation may return “No records found in input data”
  - If the workflow errors before the Respond to Webhook node runs, the caller may receive an error instead of the structured response
- **Sub-workflow reference:** none

---

## Block 2 — Mapping Configuration

### Overview
This block defines the transformation contract. It centralizes all mapping and conversion rules in one node so users can modify behavior without touching JavaScript code.

### Nodes Involved
- Configure Field Mapping

### Node Details

#### Configure Field Mapping
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a JSON configuration object containing field mappings and global settings.
- **Configuration choices:**
  - Mode: raw JSON output
  - Outputs:
    - `fieldMappings`: array of mapping rules
    - `globalSettings`: conversion and normalization options
- **Key expressions or variables used:**
  - No expressions; the node emits static JSON
  - Downstream code references:
    - `$('Configure Field Mapping').first().json`
- **Configured field mappings:**
  1. `Artikelnr` → `article_number`
     - type: `string`
     - default: empty string
     - required: `true`
  2. `Bezeichnung` → `description`
     - type: `string`
     - default: empty string
     - required: `true`
  3. `Preis` → `price`
     - type: `number`
     - default: `"0"`
     - required: `true`
  4. `Gewicht_kg` → `weight_kg`
     - type: `number`
     - default: `"0"`
     - required: `false`
  5. `Aktiv` → `is_active`
     - type: `boolean`
     - default: `"true"`
     - required: `false`
  6. `Erstellt_am` → `created_at`
     - type: `date`
     - default: empty string
     - required: `false`
- **Configured global settings:**
  - `removeUnmappedFields: true`
  - `trimStrings: true`
  - `emptyStringToNull: true`
  - `dateInputFormat: DD.MM.YYYY`
  - `dateOutputFormat: YYYY-MM-DD`
  - `decimalSeparator: ,`
- **Input and output connections:**
  - Input: `Receive Data`
  - Output: `Transform Records`
- **Version-specific requirements:**
  - Uses node type version `3.4`
  - Raw JSON mode requires valid JSON syntax; invalid formatting will break execution
- **Edge cases or potential failure types:**
  - Invalid JSON in `jsonOutput` prevents the node from executing
  - Misconfigured mapping rules may cause missing fields, failed conversions, or misleading validation
  - `defaultValue` values are strings in the config; the Code node converts them based on `dataType`
- **Sub-workflow reference:** none

---

## Block 3 — Record Transformation and Validation

### Overview
This is the core logic block. It normalizes the incoming payload, loops through each record, applies configurable transformations, and produces a single summary response object.

### Nodes Involved
- Transform Records

### Node Details

#### Transform Records
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes custom JavaScript to transform and validate records.
- **Configuration choices:**
  - JavaScript mode
  - Returns a single item with a JSON summary object
- **Key expressions or variables used:**
  - Input payload:
    - `const raw = $('Receive Data').first().json.body || $('Receive Data').first().json;`
  - Configuration:
    - `const config = $('Configure Field Mapping').first().json;`
  - Extracted values:
    - `const mappings = config.fieldMappings || [];`
    - `const settings = config.globalSettings || {};`
- **Main logic performed:**
  1. Detects input shape:
     - `{ records: [...] }`
     - raw array
     - single object
  2. Wraps a single object into an array
  3. Returns a failure response if no records exist
  4. Defines helpers:
     - `parseDate`
     - `parseNumber`
     - `parseBoolean`
  5. Iterates through records and mapping definitions
  6. Applies:
     - trimming
     - empty-string normalization
     - defaults
     - required checks
     - data type conversion
  7. Removes or preserves unmapped fields based on config
  8. Aggregates all row errors
  9. Calculates:
     - `success`
     - `totalRecords`
     - `validRecords`
     - `errorCount`
     - `records`
     - `errors`
- **Supported type conversions:**
  - `string`: coerces via `String(value)`
  - `number`: locale-aware handling of decimal separator and thousand separators
  - `boolean`: accepts values such as `true`, `1`, `yes`, `ja`, `aktiv`, `on`, and corresponding false values
  - `date`: parses based on configured input format and rewrites into configured output format
  - `auto`: effectively leaves value unchanged
- **Input and output connections:**
  - Input: `Configure Field Mapping`
  - Output: `Return Transformed Data`
- **Version-specific requirements:**
  - Uses node type version `2`
  - Uses `$('<node name>').first()` references, which depend on stable node names; renaming nodes requires updating the code
- **Edge cases or potential failure types:**
  - If `Receive Data` or `Configure Field Mapping` are renamed, the node breaks unless references are updated
  - Date parsing only performs basic day/month range checks; invalid dates like `31.02.2026` may still format instead of being rejected
  - Number parsing assumes one decimal convention at a time based on `decimalSeparator`
  - Boolean parsing is limited to the hardcoded accepted terms
  - If `removeUnmappedFields` is `true`, any unexpected source fields are dropped
  - If all errors belong to unique rows, `validRecords` is calculated as total rows minus the number of rows with at least one error
  - If an input record is malformed or nested fields are expected, the current logic does not support nested-path mapping
- **Sub-workflow reference:** none

---

## Block 4 — HTTP Response

### Overview
This block sends the final transformation result back to the caller. It closes the webhook request with a JSON response body.

### Nodes Involved
- Return Transformed Data

### Node Details

#### Return Transformed Data
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the final HTTP response for a webhook running in `responseNode` mode.
- **Configuration choices:**
  - Respond with: `json`
  - Response body: `={{ JSON.stringify($json) }}`
  - Response header:
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:**
  - Input: `Transform Records`
  - Output: none
- **Version-specific requirements:**
  - Uses node type version `1.5`
  - Must be paired with a Webhook node configured for `responseNode`
- **Edge cases or potential failure types:**
  - If upstream node output is not serializable, response generation could fail
  - Since `respondWith` is already set to JSON, stringifying manually may be redundant; however, it still returns a JSON-formatted payload
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note describing the overall workflow, setup, customization, and author information |  |  | ## Transform and Map Data Fields, Standalone, No External Services |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note explaining webhook input format |  |  | ### Step 1: Receive Data |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note explaining the mapping configuration structure |  |  | ### Step 2: Configure Field Mapping |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note describing transformation logic and output shape per record |  |  | ### Step 3: Transform Records |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note explaining final response structure and downstream usage |  |  | ### Step 4: Return Result |
| Receive Data | n8n-nodes-base.webhook | Receives POSTed source records through an HTTP endpoint |  | Configure Field Mapping | ## Transform and Map Data Fields, Standalone, No External Services  |
| Receive Data | n8n-nodes-base.webhook | Receives POSTed source records through an HTTP endpoint |  | Configure Field Mapping | ### Step 1: Receive Data |
| Configure Field Mapping | n8n-nodes-base.set | Stores field mapping rules and global transformation settings | Receive Data | Transform Records | ## Transform and Map Data Fields, Standalone, No External Services  |
| Configure Field Mapping | n8n-nodes-base.set | Stores field mapping rules and global transformation settings | Receive Data | Transform Records | ### Step 2: Configure Field Mapping |
| Transform Records | n8n-nodes-base.code | Applies mappings, type conversion, defaults, validation, and error aggregation | Configure Field Mapping | Return Transformed Data | ## Transform and Map Data Fields, Standalone, No External Services  |
| Transform Records | n8n-nodes-base.code | Applies mappings, type conversion, defaults, validation, and error aggregation | Configure Field Mapping | Return Transformed Data | ### Step 3: Transform Records |
| Return Transformed Data | n8n-nodes-base.respondToWebhook | Returns the final JSON response to the webhook caller | Transform Records |  | ## Transform and Map Data Fields, Standalone, No External Services  |
| Return Transformed Data | n8n-nodes-base.respondToWebhook | Returns the final JSON response to the webhook caller | Transform Records |  | ### Step 4: Return Result |

Notes for the table:
- The large overview sticky note visually covers the whole functional workflow area, so its content is duplicated for each affected execution node.
- Step-specific sticky notes are also duplicated on the nodes they describe.

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: `Transform and map data fields standalone via webhook with configurable type conversion`.

2. **Add the Webhook node**
   - Create a **Webhook** node.
   - Rename it to **Receive Data**.
   - Set:
     - **HTTP Method**: `POST`
     - **Path**: `transform-data`
     - **Response Mode**: `Using Respond to Webhook Node` / `responseNode`
   - Leave other options at default unless you need custom authentication or response behavior.

3. **Add the Set node for configuration**
   - Create a **Set** node.
   - Rename it to **Configure Field Mapping**.
   - Connect **Receive Data** → **Configure Field Mapping**.
   - Set the node to output raw JSON.
   - Paste this configuration conceptually as valid JSON:
     - `fieldMappings` array with these entries:
       - `Artikelnr` → `article_number`, type `string`, default `""`, required `true`
       - `Bezeichnung` → `description`, type `string`, default `""`, required `true`
       - `Preis` → `price`, type `number`, default `"0"`, required `true`
       - `Gewicht_kg` → `weight_kg`, type `number`, default `"0"`, required `false`
       - `Aktiv` → `is_active`, type `boolean`, default `"true"`, required `false`
       - `Erstellt_am` → `created_at`, type `date`, default `""`, required `false`
     - `globalSettings`:
       - `removeUnmappedFields: true`
       - `trimStrings: true`
       - `emptyStringToNull: true`
       - `dateInputFormat: "DD.MM.YYYY"`
       - `dateOutputFormat: "YYYY-MM-DD"`
       - `decimalSeparator: ","`
   - Ensure the JSON is syntactically valid.

4. **Add the Code node**
   - Create a **Code** node.
   - Rename it to **Transform Records**.
   - Connect **Configure Field Mapping** → **Transform Records**.
   - Set it to JavaScript mode.
   - Add logic that:
     1. Reads the inbound webhook payload from **Receive Data**
     2. Reads mapping config from **Configure Field Mapping**
     3. Accepts:
        - `{ records: [...] }`
        - raw arrays
        - single objects
     4. Converts single objects into a one-item array
     5. Returns an error object if no records are found
     6. Defines helper functions for:
        - date parsing by configurable format
        - number parsing with configurable decimal separator
        - boolean parsing with multilingual values like `ja/nein`
     7. Iterates through every record and every mapping
     8. For each mapped field:
        - gets source value
        - trims strings if enabled
        - converts empty strings to `null` if enabled
        - applies default values if value is empty
        - validates required fields
        - converts by target type
     9. Optionally removes unmapped fields when `removeUnmappedFields` is `true`
     10. Aggregates all row-level errors
     11. Returns one JSON object with:
        - `success`
        - `totalRecords`
        - `validRecords`
        - `errorCount`
        - `records`
        - `errors`
   - Important: if you use node-name references in code such as `$('Receive Data')`, keep those node names exactly as written or update the code accordingly.

5. **Use the transformation logic from the workflow**
   - The Code node should implement these key helper behaviors:
     - **parseNumber**
       - If decimal separator is `,`, remove dots as thousand separators and replace comma with dot before calling `Number(...)`
       - If decimal separator is `.`, remove commas as thousand separators
     - **parseBoolean**
       - True values accepted: `true`, `1`, `yes`, `ja`, `y`, `j`, `aktiv`, `active`, `on`
       - False values accepted: `false`, `0`, `no`, `nein`, `n`, `inaktiv`, `inactive`, `off`
     - **parseDate**
       - Interpret input by configured pattern such as `DD.MM.YYYY`
       - Output a reformatted string such as `YYYY-MM-DD`
   - The code should also calculate valid records by counting distinct rows without any errors.

6. **Add the Respond to Webhook node**
   - Create a **Respond to Webhook** node.
   - Rename it to **Return Transformed Data**.
   - Connect **Transform Records** → **Return Transformed Data**.
   - Set:
     - **Respond With**: `JSON`
     - **Response Body**: `={{ JSON.stringify($json) }}`
   - Add response header:
     - `Content-Type: application/json`

7. **Activate the workflow**
   - Save the workflow.
   - Activate it so the production webhook URL becomes available.

8. **Test with sample input**
   - Send a `POST` request to the webhook path `/transform-data`.
   - Example payload:
     ```json
     {
       "records": [
         {
           "Artikelnr": "A-1001",
           "Bezeichnung": "Schraube M8",
           "Preis": "12,50",
           "Gewicht_kg": "",
           "Aktiv": "ja",
           "Erstellt_am": "15.03.2026"
         }
       ]
     }
     ```
   - Expected transformed result:
     - `article_number`: `"A-1001"`
     - `description`: `"Schraube M8"`
     - `price`: `12.5`
     - `weight_kg`: `0`
     - `is_active`: `true`
     - `created_at`: `"2026-03-15"`

9. **Understand default behavior**
   - If an optional field is empty and has a non-empty default, the default is used.
   - If a required field remains empty after default handling, an error is generated.
   - If conversion fails, the target field is set to `null` and an error is added.
   - If `removeUnmappedFields` is `true`, only mapped target fields remain in output.

10. **No credentials are required**
    - This workflow uses no external APIs or authentication providers.
    - No OAuth, API key, or database credentials are needed.

11. **Optional downstream extension**
    - If you want to save or forward the transformed result:
      - Keep **Return Transformed Data** as the webhook response endpoint
      - Add separate processing logic before the response if you need synchronous downstream handling
      - Or duplicate/transmit the transformed data elsewhere depending on your design
    - If you place downstream nodes after the response node, verify your n8n execution pattern and whether you want processing to continue after responding.

12. **Optional customization points**
    - Add more entries to `fieldMappings`
    - Change `dataType` to `auto` if a field should be passed through without forced conversion
    - Set `removeUnmappedFields` to `false` to preserve extra source fields
    - Change locale settings for:
      - `decimalSeparator`
      - `dateInputFormat`
      - `dateOutputFormat`
    - Expand boolean synonyms inside the Code node if your source systems use different labels

13. **Recommended implementation caution**
    - Keep node names stable because the Code node depends on them
    - If using non-flat JSON input, extend the code to support nested field paths
    - If strict calendar validation is required, improve the date parser to reject impossible dates like `31.02.2026`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: Florian Eiche | Workflow note |
| eiche-digital.de | https://eiche-digital.de |
| The workflow is fully self-contained and uses no external services or credentials. | General architecture |
| Intended use: normalize imported records before loading them into ERP, CRM, databases, or APIs. | Use case |
| Input supports both `{ records: [...] }` and a single object payload. | Webhook contract |
| Locale-aware conversions include comma decimals, German-style dates, and `ja/nein` boolean parsing. | Data normalization behavior |