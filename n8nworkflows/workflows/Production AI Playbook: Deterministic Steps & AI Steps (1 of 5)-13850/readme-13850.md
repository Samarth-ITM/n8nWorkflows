Production AI Playbook: Deterministic Steps & AI Steps (1 of 5)

https://n8nworkflows.xyz/workflows/production-ai-playbook--deterministic-steps---ai-steps--1-of-5--13850


# Production AI Playbook: Deterministic Steps & AI Steps (1 of 5)

# 1. Workflow Overview

This workflow implements the first stage of a production-oriented AI intake pattern: it receives inbound lead data, normalizes inconsistent payload formats, validates required fields deterministically, and returns either a success acknowledgment or a structured validation error.

Its main purpose is to ensure that only clean, well-formed lead records are passed downstream to later AI or automation stages. In this workflow, the “AI” step is not yet executed; instead, the workflow prepares data so later workflows can consume it safely.

## 1.1 Input Reception

The workflow starts with a webhook that accepts incoming lead data via HTTP POST. The request is expected to contain a JSON body with lead-related fields such as name, email, company, and message.

## 1.2 Input Normalization

A Code node standardizes multiple possible input field names into a single canonical structure. It also trims values, lowercases email and source fields, fills defaults where needed, and computes an initial `isValid` flag.

## 1.3 Deterministic Validation Routing

An IF node reads the `isValid` flag and splits the flow into valid and invalid branches. This deterministic gate prevents malformed inputs from reaching downstream processing.

## 1.4 Success Preparation

If the payload is valid, a Code node enriches the record with processing metadata and marks it as ready for downstream AI processing. A webhook response node then returns a success JSON response.

## 1.5 Error Preparation

If the payload is invalid, a Code node computes explicit validation error messages and attaches them to the payload. A webhook response node then returns an HTTP 400 response with a structured error object.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block exposes the workflow as an HTTP endpoint. It receives external lead submissions and passes them into the normalization stage without responding immediately; the response is deferred to dedicated response nodes later in the workflow.

### Nodes Involved
- Webhook - Receive Lead Data

### Node Details

#### Webhook - Receive Lead Data
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that listens for incoming HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `incoming-lead`
  - Response mode: `responseNode`, meaning the workflow will respond later using Respond to Webhook nodes
- **Key expressions or variables used:**
  - No dynamic expressions configured in this node
  - Downstream nodes rely on request payload under `$json.body`
- **Input and output connections:**
  - No input node; this is an entry point
  - Outputs to: `Normalize Input Data`
- **Version-specific requirements:**
  - Uses node type version `2`
  - `responseNode` mode requires at least one reachable Respond to Webhook node in each execution path
- **Edge cases or potential failure types:**
  - If no response node is reached, the request can hang or fail
  - If the caller sends a payload format inconsistent with expected JSON body structure, downstream code may not find fields
  - Incorrect webhook activation or test/production URL usage can cause apparent endpoint failure
- **Sub-workflow reference:** None

---

## Block 2 — Input Normalization

### Overview
This block transforms inconsistent inbound lead payloads into a stable schema. It also performs lightweight initial validation so later routing can rely on a single boolean flag instead of repeatedly checking raw fields.

### Nodes Involved
- Normalize Input Data

### Node Details

#### Normalize Input Data
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node used to map raw webhook input into a normalized lead object.
- **Configuration choices:**
  - Reads the first input item via `$input.first().json.body`
  - Builds a normalized object with canonical keys:
    - `name`
    - `email`
    - `company`
    - `message`
    - `source`
    - `receivedAt`
  - Supports alternate field names:
    - Name: `fullName`, `name`, `full_name`
    - Email: `email`, `emailAddress`, `email_address`
    - Company: `company`, `organization`, `org`
    - Message: `message`, `body`, `content`, `inquiry`
    - Source: `source`, `utm_source`
  - Applies cleanup:
    - `.trim()` on text fields
    - `.toLowerCase()` on email and source
    - default `company = 'Unknown'`
    - default `source = 'web'`
  - Computes `isValid` using:
    - email exists and includes `@`
    - name exists
    - message exists
- **Key expressions or variables used:**
  - `$input.first().json.body`
  - `new Date().toISOString()`
  - Local variables: `raw`, `normalized`
- **Input and output connections:**
  - Input from: `Webhook - Receive Lead Data`
  - Output to: `Validate Required Fields`
- **Version-specific requirements:**
  - Uses Code node version `2`
  - Assumes JavaScript execution environment supports modern syntax used in the script
- **Edge cases or potential failure types:**
  - If `$json.body` is undefined, expressions like `(raw.fullName || ...)` will throw because `raw` becomes undefined
  - If fields arrive in unexpected nested structures, normalization will miss them
  - `email.includes('@')` is a minimal validity check and may accept malformed addresses
  - Numeric or non-string values may fail if `.trim()` is applied to non-strings; this is avoided only if values are absent or already strings
- **Sub-workflow reference:** None

---

## Block 3 — Deterministic Validation Routing

### Overview
This block decides whether the normalized lead can proceed. It uses the previously computed `isValid` flag as a deterministic gate and separates success handling from error handling.

### Nodes Involved
- Validate Required Fields

### Node Details

#### Validate Required Fields
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router that branches based on whether the normalized lead is considered valid.
- **Configuration choices:**
  - Condition type uses n8n IF node version 2 style condition definitions
  - Evaluates whether `{{$json.isValid}}` is boolean `true`
  - Combinator is `and`, though only one condition is currently configured
- **Key expressions or variables used:**
  - `={{ $json.isValid }}`
- **Input and output connections:**
  - Input from: `Normalize Input Data`
  - True output to: `Valid Lead - Ready for AI`
  - False output to: `Invalid Lead - Error Handler`
- **Version-specific requirements:**
  - Uses IF node version `2.2`
  - Strict type validation is enabled in condition options; this matters if `isValid` is not a true boolean
- **Edge cases or potential failure types:**
  - If upstream code sets `isValid` as string `"true"` instead of boolean `true`, the condition will fail because strict typing is enabled
  - If upstream data is malformed and `isValid` is absent, the item will route to the false branch
- **Sub-workflow reference:** None

---

## Block 4 — Valid Lead Preparation and Success Response

### Overview
This block handles accepted leads. It adds metadata confirming successful validation, marks the item as ready for downstream AI processing, and sends a success HTTP response to the caller.

### Nodes Involved
- Valid Lead - Ready for AI
- Respond - Success

### Node Details

#### Valid Lead - Ready for AI
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that enriches validated lead data with processing metadata for downstream use.
- **Configuration choices:**
  - Reads the first input item
  - Returns the original normalized lead plus:
    - `isValid: true`
    - `processedAt: <ISO timestamp>`
    - `readyForAI: true`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from: `Validate Required Fields` true branch
  - Output to: `Respond - Success`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If no input item is present, `$input.first()` can fail
  - This node does not itself call AI or another workflow; it only tags the item for future downstream AI usage
- **Sub-workflow reference:** None

#### Respond - Success
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the final HTTP response for valid requests.
- **Configuration choices:**
  - Responds with JSON
  - Response body expression returns:
    - `{ status: 'received', valid: true }`
  - No custom status code configured, so the standard success code is used
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ status: 'received', valid: true }) }}`
- **Input and output connections:**
  - Input from: `Valid Lead - Ready for AI`
  - No output nodes
- **Version-specific requirements:**
  - Uses Respond to Webhook version `1.1`
- **Edge cases or potential failure types:**
  - If upstream path does not originate from a webhook in response-node mode, this node cannot behave as intended
  - Returning a stringified JSON body while `respondWith` is `json` may be acceptable in n8n, but some builders prefer returning an object directly to avoid ambiguity
- **Sub-workflow reference:** None

---

## Block 5 — Invalid Lead Handling and Error Response

### Overview
This block generates explicit validation feedback for malformed requests. Instead of silently rejecting input, it produces a structured list of failed fields and returns it with HTTP status 400.

### Nodes Involved
- Invalid Lead - Error Handler
- Respond - Validation Error

### Node Details

#### Invalid Lead - Error Handler
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that recomputes field-level validation errors and packages them for API response.
- **Configuration choices:**
  - Reads normalized lead data from the first input item
  - Initializes an `errors` array
  - Adds messages for:
    - missing or invalid email
    - missing name
    - missing message
  - Returns the original lead plus:
    - `isValid: false`
    - `validationErrors: errors`
    - `processedAt: <ISO timestamp>`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from: `Validate Required Fields` false branch
  - Output to: `Respond - Validation Error`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If normalization failed earlier and fields are missing entirely, this node still works as long as an input item exists
  - Email validation remains intentionally simple and may not catch advanced invalid formats
- **Sub-workflow reference:** None

#### Respond - Validation Error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a client-facing error response for invalid submissions.
- **Configuration choices:**
  - Response code: `400`
  - Responds with JSON
  - Response body expression returns:
    - `{ status: 'error', errors: $json.validationErrors }`
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ status: 'error', errors: $json.validationErrors }) }}`
- **Input and output connections:**
  - Input from: `Invalid Lead - Error Handler`
  - No output nodes
- **Version-specific requirements:**
  - Uses Respond to Webhook version `1.1`
- **Edge cases or potential failure types:**
  - If `validationErrors` is absent, the response still returns but with an undefined or empty error list
  - Same note as the success response: stringifying JSON while using JSON response mode may be acceptable but is not the only possible implementation
- **Sub-workflow reference:** None

---

## Block 6 — Documentation / In-Canvas Notes

### Overview
These nodes are informational only and do not affect execution. They explain how the workflow works, how to test it, and where it fits in the broader Production AI Playbook series.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation node covering the overall workflow purpose, setup, and customization guidance.
- **Configuration choices:**
  - Contains a long note describing normalization, validation, testing, customization, and a blog link
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None; non-executable
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Section header for the receive/normalize area.
- **Configuration choices:**
  - Label: `Receive & Normalize`
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Section header for the validation area.
- **Configuration choices:**
  - Label: `Validate`
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Receive Lead Data | Webhook | Receives POST lead submissions and starts execution |  | Normalize Input Data | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Receive & Normalize |
| Normalize Input Data | Code | Maps raw request body into canonical lead fields and computes `isValid` | Webhook - Receive Lead Data | Validate Required Fields | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Receive & Normalize |
| Validate Required Fields | If | Routes normalized lead to valid or invalid branch | Normalize Input Data | Valid Lead - Ready for AI; Invalid Lead - Error Handler | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Validate |
| Valid Lead - Ready for AI | Code | Adds metadata and marks valid lead as ready for downstream AI | Validate Required Fields | Respond - Success | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Validate |
| Respond - Success | Respond to Webhook | Returns success JSON response for valid lead submissions | Valid Lead - Ready for AI |  | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Validate |
| Invalid Lead - Error Handler | Code | Builds explicit validation error messages for invalid leads | Validate Required Fields | Respond - Validation Error | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Validate |
| Respond - Validation Error | Respond to Webhook | Returns HTTP 400 with validation error list | Invalid Lead - Error Handler |  | ## Input Normalization + Validation<br>### How it works<br>1. **Webhook** receives raw lead data via POST request.<br>2. **Normalize Input Data** (Code node) standardizes field names, trims whitespace, lowercases emails, and flattens inconsistent payloads into a clean object.<br>3. **Validate Required Fields** (IF node) checks that email is valid and message is non-empty. Valid leads continue; invalid leads route to error handling.<br>4. **Valid Lead** path packages the cleaned data, ready for downstream AI processing. **Invalid Lead** path returns a structured error response listing which fields failed.<br>### Setup<br>- Import the template and open the **Webhook** node to copy its test URL<br>- Use a tool like curl or Postman to send a POST request with JSON lead data (fields: name, email, company, message)<br>- Inspect the **Normalize Input Data** code node to customize field mappings for your data sources<br>### Customization<br>- Add more validation rules in the IF node (e.g., check company against a known list)<br>- Extend normalization to handle additional input formats or sources<br>- Connect the Valid Lead output to your AI classification or enrichment workflow<br>This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n.<br>https://go.n8n.io/PAP-D&A-Blog<br>## Validate |
| Sticky Note | Sticky Note | In-canvas documentation for setup, behavior, and customization |  |  |  |
| Sticky Note1 | Sticky Note | Section title for receive/normalize area |  |  |  |
| Sticky Note2 | Sticky Note | Section title for validation area |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it: `Production AI Playbook: Deterministic Steps & AI Steps (1 of 5)`.

2. **Add a Webhook node**.
   - Name: `Webhook - Receive Lead Data`
   - Type: `Webhook`
   - Set:
     - **HTTP Method**: `POST`
     - **Path**: `incoming-lead`
     - **Response Mode**: `Using Respond to Webhook Node` / `responseNode`
   - This node will serve as the entry point.

3. **Add a Code node** connected from the Webhook node.
   - Name: `Normalize Input Data`
   - Type: `Code`
   - Configure it to run JavaScript that reads the request body and maps alternate field names into a canonical object.
   - Use logic equivalent to:
     - Read: `$input.first().json.body`
     - Build:
       - `name` from `fullName`, `name`, or `full_name`
       - `email` from `email`, `emailAddress`, or `email_address`
       - `company` from `company`, `organization`, or `org`, defaulting to `Unknown`
       - `message` from `message`, `body`, `content`, or `inquiry`
       - `source` from `source` or `utm_source`, defaulting to `web`
       - `receivedAt` as current ISO timestamp
     - Normalize:
       - trim text
       - lowercase email and source
     - Compute:
       - `isValid = email exists and contains '@' AND name exists AND message exists`
   - Return the normalized object as JSON.

4. **Add an IF node** connected from `Normalize Input Data`.
   - Name: `Validate Required Fields`
   - Type: `If`
   - Create one condition:
     - Left value: `{{$json.isValid}}`
     - Operator: `is true`
   - Keep strict type behavior so the node expects a real boolean.

5. **Create the valid branch Code node** from the IF node’s true output.
   - Name: `Valid Lead - Ready for AI`
   - Type: `Code`
   - Configure JavaScript to:
     - read the incoming normalized lead
     - return all existing fields plus:
       - `isValid: true`
       - `processedAt: new Date().toISOString()`
       - `readyForAI: true`

6. **Add a Respond to Webhook node** after the valid branch.
   - Name: `Respond - Success`
   - Type: `Respond to Webhook`
   - Set:
     - **Respond With**: `JSON`
     - **Response Body**: an expression equivalent to  
       `{{ JSON.stringify({ status: 'received', valid: true }) }}`
   - Leave default success HTTP code unless you specifically want to set `200`.

7. **Create the invalid branch Code node** from the IF node’s false output.
   - Name: `Invalid Lead - Error Handler`
   - Type: `Code`
   - Configure JavaScript to:
     - read the normalized lead
     - initialize an empty `errors` array
     - push `Missing or invalid email` if no valid email-like string exists
     - push `Missing name` if name is empty
     - push `Missing message` if message is empty
     - return all original fields plus:
       - `isValid: false`
       - `validationErrors: errors`
       - `processedAt: new Date().toISOString()`

8. **Add a Respond to Webhook node** after the invalid branch.
   - Name: `Respond - Validation Error`
   - Type: `Respond to Webhook`
   - Set:
     - **Respond With**: `JSON`
     - **Response Code**: `400`
     - **Response Body**: an expression equivalent to  
       `{{ JSON.stringify({ status: 'error', errors: $json.validationErrors }) }}`

9. **Connect the nodes in this order**:
   - `Webhook - Receive Lead Data` → `Normalize Input Data`
   - `Normalize Input Data` → `Validate Required Fields`
   - `Validate Required Fields` true → `Valid Lead - Ready for AI`
   - `Valid Lead - Ready for AI` → `Respond - Success`
   - `Validate Required Fields` false → `Invalid Lead - Error Handler`
   - `Invalid Lead - Error Handler` → `Respond - Validation Error`

10. **Add optional Sticky Note nodes** for documentation.
    - One large note describing:
      - how the webhook receives input
      - how normalization works
      - how validation routes leads
      - how to test with curl or Postman
      - how to extend the workflow for downstream AI classification or enrichment
      - include the link: `https://go.n8n.io/PAP-D&A-Blog`
    - One smaller note labeled `Receive & Normalize`
    - One smaller note labeled `Validate`

11. **Test the workflow with a valid payload**.
    - Example body:
      ```json
      {
        "name": "Jane Doe",
        "email": "jane@example.com",
        "company": "Acme",
        "message": "I would like a demo"
      }
      ```
    - Expected response:
      ```json
      {
        "status": "received",
        "valid": true
      }
      ```

12. **Test the workflow with an alternate field naming pattern**.
    - Example body:
      ```json
      {
        "full_name": "Jane Doe",
        "emailAddress": "jane@example.com",
        "organization": "Acme",
        "content": "Please contact me",
        "utm_source": "LinkedIn"
      }
      ```
    - This should still normalize successfully.

13. **Test the workflow with an invalid payload**.
    - Example body:
      ```json
      {
        "name": "",
        "email": "invalid-email",
        "message": ""
      }
      ```
    - Expected HTTP status: `400`
    - Expected response body similar to:
      ```json
      {
        "status": "error",
        "errors": [
          "Missing or invalid email",
          "Missing name",
          "Missing message"
        ]
      }
      ```

14. **Activate the workflow** once testing is complete.
    - Use the production webhook URL for real inbound requests.

15. **Optional hardening improvements if rebuilding for production**:
    - In `Normalize Input Data`, guard against missing request body with a fallback such as an empty object
   - Replace the basic email check with stricter validation if needed
   - Add rate limiting or API authentication upstream
   - Forward the valid branch to an AI enrichment or classification workflow after `Valid Lead - Ready for AI`

### Credentials Configuration
- **No credentials are required** for the current workflow.
- There is no OpenAI, email, database, or SaaS credential used in this version.

### Sub-workflow Setup
- **No sub-workflows are used**.
- No Execute Workflow node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This template is a learning companion to the Production AI Playbook, a series that explores strategies, shares best practices, and provides practical examples for building reliable AI systems in n8n. | In-canvas documentation |
| Blog/resource link referenced in the workflow notes | https://go.n8n.io/PAP-D&A-Blog |
| Suggested testing tools include curl and Postman for sending POST requests to the webhook. | Setup/testing guidance |
| The workflow is designed as a deterministic validation layer before downstream AI classification or enrichment. | Architectural context |