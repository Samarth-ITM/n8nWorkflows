Validate JSON payloads against a schema with detailed error messages (no AI)

https://n8nworkflows.xyz/workflows/validate-json-payloads-against-a-schema-with-detailed-error-messages--no-ai--14208


# Validate JSON payloads against a schema with detailed error messages (no AI)

## 1. Workflow Overview

This workflow is a reusable **JSON payload validation sub-workflow** for n8n. Its core purpose is to accept:

- a JSON Schema-like object in `requiredSchema`
- a JSON payload in `paramsToValidate`

and return either:

- `{ valid: true, validationError: null }`, or
- `{ valid: false, validationError: "...detailed message...", requiredSchema: {...} }`

It does **not** use AI at runtime. Instead, it relies on a custom JavaScript validator implemented in a Code node. The canvas also includes:
- documentation sticky notes
- a schema-generation prompt helper area for LLM usage
- a usage example showing how to integrate the validator into a webhook flow

### 1.1 Core Validation Sub-Workflow
This is the actual functional validator. It starts when called by another workflow and runs the custom validation logic.

**Nodes:**
- `When Executed by Another Workflow`
- `Schema Validation`

### 1.2 Schema Generation Prompt Helper
This is a non-executing helper area meant to be copied into an LLM chat so the model can generate a compatible schema or ready-to-paste n8n nodes.

**Nodes:**
- `Sticky Note1`
- `Sticky Note2`
- `PLACEHOLDER: Source of your data`
- `Call 'Param Schema Validation Template'`

### 1.3 Webhook Integration Example
This is a demonstration flow showing how to use the validator from a webhook endpoint, branch on `valid`, and return a 400 response when validation fails.

**Nodes:**
- `Sticky Note3`
- `Webhook`
- `Validate Schema`
- `If Params Valid`
- `Return 400 param error`
- `Your workflow logic here`
- `Return Success Response`
- `Sticky Note4`

### 1.4 Canvas Documentation
These sticky notes explain what the workflow does and identify which nodes are the real implementation versus examples/documentation.

**Nodes:**
- `Sticky Note`
- `Sticky Note5`

---

## 2. Block-by-Block Analysis

## 2.1 Core Validation Sub-Workflow

### Overview
This block is the real implementation. It exposes two workflow inputs and validates the provided payload against the provided schema using a custom validator written in JavaScript.

### Nodes Involved
- `When Executed by Another Workflow`
- `Schema Validation`

### Node Details

#### 2.1.1 When Executed by Another Workflow
- **Type and technical role:** `n8n-nodes-base.executeWorkflowTrigger`  
  Entry point for a sub-workflow invoked by an Execute Sub-Workflow / Execute Workflow node.
- **Configuration choices:**
  - Declares two workflow inputs:
    - `requiredSchema` as `object`
    - `paramsToValidate` as `object`
- **Key expressions or variables used:**
  - No expressions inside the node itself.
  - These inputs are later read in the Code node.
- **Input and output connections:**
  - No incoming connection; this is an entry node.
  - Outputs to `Schema Validation`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
  - Requires n8n support for workflow input definitions on sub-workflow trigger nodes.
- **Edge cases or potential failure types:**
  - Caller may pass malformed data or a string instead of an object expression.
  - If the parent workflow does not map the inputs correctly, the downstream validator may return an invalid-schema or missing-data error.
- **Sub-workflow reference:**
  - This node belongs to the sub-workflow itself and is the callable entry point.

#### 2.1.2 Schema Validation
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes a custom JavaScript JSON Schema validator and formats detailed human-readable error output.
- **Configuration choices:**
  - Reads:
    - `const requiredSchema = $input.first().json.requiredSchema;`
    - `const jsonToValidate = $input.first().json.paramsToValidate;`
  - Returns one object containing:
    - `valid`
    - `validationError`
    - optionally `requiredSchema`
  - Wrapped in `try/catch` to convert runtime exceptions into a `Validator error: ...` response.
- **Key expressions or variables used:**
  - `$input.first().json.requiredSchema`
  - `$input.first().json.paramsToValidate`
  - Internal helper functions include:
    - `makeError`
    - `getTypeName`
    - `isInteger`
    - `isValidNumber`
    - `isPlainObject`
    - `deepEqual`
    - `safeRegexTest`
    - `formatFieldList`
    - `buildWhenClause`
    - `isNullAllowed`
    - `joinPath`
    - `indexPath`
    - `checkType`
    - `checkString`
    - `checkNumber`
    - `checkEnum`
    - `checkConst`
    - `checkArray`
    - `checkObject`
    - `checkNot`
    - `checkOneOf`
    - `checkAnyOf`
    - `checkAllOf`
    - `validate`
    - `formatError`
    - `cleanErrors`
- **Input and output connections:**
  - Input from `When Executed by Another Workflow`
  - No downstream node in this template’s core path; output is returned to the caller workflow
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Assumes modern n8n Code node JavaScript support
- **Edge cases or potential failure types:**
  - Invalid schema shape: if `requiredSchema` is not a plain object, returns `Invalid schema: expected a plain object...`
  - Null or undefined payloads: returns `Field is missing or null` unless null is explicitly allowed by `const` or `enum`
  - Invalid regex in `pattern`: does not crash; returns `Invalid regex pattern: ...`
  - Wrong `required` format: if `required` is not an array, returns schema error
  - `oneOf` edge cases:
    - empty `oneOf` => nothing can match
    - multiple matches => ambiguous
    - no matches => surfaces best-fit variant errors
  - Numeric edge cases:
    - rejects `NaN`
    - rejects `Infinity` / `-Infinity`
    - distinguishes integer vs non-integer number
  - Additional properties:
    - if `additionalProperties: false`, undeclared fields are rejected
  - Runtime JS exception:
    - converted to `valid: false` with `Validator error: ...`
- **Sub-workflow reference:**
  - This is the implementation node of the callable validation sub-workflow.

---

## 2.2 Schema Generation Prompt Helper

### Overview
This block is not part of runtime validation. It exists to help a human or AI agent generate a schema compatible with the validator, using a sample payload and a sub-workflow call example.

### Nodes Involved
- `Sticky Note1`
- `Sticky Note2`
- `PLACEHOLDER: Source of your data`
- `Call 'Param Schema Validation Template'`

### Node Details

#### 2.2.1 Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation block introducing schema creation via AI.
- **Configuration choices:**
  - Explains that AI can generate schemas.
  - Instructs the user to copy the sticky plus the two nodes below into an agent.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** Refers to the validator workflow conceptually.

#### 2.2.2 Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Large prompt template containing detailed instructions for generating compatible schemas or full n8n nodes.
- **Configuration choices:**
  - Documents supported schema keywords:
    - types, string rules, number rules, enum/const, arrays, objects, logic combinators, descriptions
  - Requires every field to include `description`
  - Explains how to embed schema in `={{ }}` expressions
  - Explains regex double escaping
  - Requests either:
    1. full n8n nodes
    2. schema only
  - Describes required response body format for validation failures
- **Key expressions or variables used:**
  - Mentions `requiredSchema`
  - Mentions `paramsToValidate`
  - Mentions `={{ $json.body }}`
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - If copied literally without adapting workflow IDs or input paths, generated nodes may fail.
- **Sub-workflow reference:**
  - Refers explicitly to the reusable validation sub-workflow.

#### 2.2.3 PLACEHOLDER: Source of your data
- **Type and technical role:** `n8n-nodes-base.set`  
  Provides a mock payload for schema-generation or demonstration purposes.
- **Configuration choices:**
  - Sets a `body` object with:
    - `email: "user@example.com"`
    - `plan: "cloud"`
    - `seat_count: 25`
- **Key expressions or variables used:** None; static object assignment
- **Input and output connections:**
  - Outputs to `Call 'Param Schema Validation Template'`
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - This is placeholder data only; if forgotten in production, it may mislead testing.
- **Sub-workflow reference:** Feeds a call to the validation sub-workflow.

#### 2.2.4 Call 'Param Schema Validation Template'
- **Type and technical role:** `n8n-nodes-base.executeWorkflow`  
  Demonstrates how another workflow would call the validator sub-workflow.
- **Configuration choices:**
  - Calls workflow ID `O1tkgab3t4olf3AJ`
  - Uses defined workflow inputs:
    - `requiredSchema`: expression-wrapped object schema
    - `paramsToValidate`: `={{ $json.body }}`
  - Example schema validates:
    - `email` as string matching email-like regex
    - `plan` as enum of `community`, `cloud`, `enterprise`
    - `seat_count` as integer 1..500
  - Required fields: `email`, `plan`
  - `additionalProperties: false`
  - Mapping mode: define below
  - `attemptToConvertTypes: false`
  - `convertFieldsToString: true`
- **Key expressions or variables used:**
  - `={{ { ...schema... } }}`
  - `={{ $json.body }}`
- **Input and output connections:**
  - Input from `PLACEHOLDER: Source of your data`
  - No outgoing connection shown
- **Version-specific requirements:**
  - Uses `typeVersion: 1.3`
  - Depends on Execute Workflow node supporting workflow input mapping
- **Edge cases or potential failure types:**
  - Workflow ID will differ in another n8n instance; user must re-select target workflow
  - If `requiredSchema` is pasted as a string instead of an expression object, validation will fail
  - Regex escaping may break if edited incorrectly
  - If `paramsToValidate` path is wrong, validator will receive null/undefined or the wrong structure
- **Sub-workflow reference:**
  - Invokes the validator workflow `Param Schema Validation Template`

---

## 2.3 Webhook Integration Example

### Overview
This block shows a practical usage pattern: receive a webhook request, validate the request body using the sub-workflow, branch on the result, and either continue business logic or return a 400 response with detailed validation feedback.

### Nodes Involved
- `Sticky Note3`
- `Webhook`
- `Validate Schema`
- `If Params Valid`
- `Return 400 param error`
- `Your workflow logic here`
- `Return Success Response`
- `Sticky Note4`

### Node Details

#### 2.3.1 Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels this section as a usage example for webhook wiring.
- **Configuration choices:**
  - Notes this is an example for wiring the validator into a webhook endpoint.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** Indirectly refers to validator usage.

#### 2.3.2 Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for HTTP requests in the example flow.
- **Configuration choices:**
  - Webhook path: `19b37d89-d47e-4f90-a945-8a92d61d8d8b`
  - `responseMode: responseNode`, meaning a downstream Respond to Webhook node must send the HTTP response
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs to `Validate Schema`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.1`
- **Edge cases or potential failure types:**
  - If no Respond to Webhook path is reached, webhook requests may time out
  - Webhook URL/path will differ after import or activation depending on environment
- **Sub-workflow reference:** Its output is validated by the sub-workflow call.

#### 2.3.3 Validate Schema
- **Type and technical role:** `n8n-nodes-base.executeWorkflow`  
  Calls the validator sub-workflow from the webhook example.
- **Configuration choices:**
  - Calls workflow ID `O1tkgab3t4olf3AJ`
  - Passes:
    - `requiredSchema` as an inline object expression
    - `paramsToValidate` as `={{ $json.body }}`
  - Example schema requires:
    - `name`: non-empty string
    - `email`: email-like regex string
    - `plan`: enum `starter`, `pro`, `enterprise`
    - `seat_count`: integer between 1 and 500
    - `tags`: array of non-empty strings, minimum 1 item
  - Required fields:
    - `name`
    - `email`
    - `plan`
    - `seat_count`
  - `additionalProperties: false`
  - Mapping mode set explicitly with workflow input schema
  - `attemptToConvertTypes: false`
  - `convertFieldsToString: true`
- **Key expressions or variables used:**
  - `={{ { ...schema... } }}`
  - `={{ $json.body }}`
- **Input and output connections:**
  - Input from `Webhook`
  - Output to `If Params Valid`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.3`
- **Edge cases or potential failure types:**
  - Imported workflow must re-bind target workflow ID
  - If body is missing, validation returns missing/null errors
  - Improperly escaped regex breaks pattern validation
  - If schema expression syntax is malformed, the node may fail before calling the sub-workflow
- **Sub-workflow reference:**
  - Invokes `Param Schema Validation Template`

#### 2.3.4 If Params Valid
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches on the validator result.
- **Configuration choices:**
  - Checks `={{ $json.valid }}`
  - Condition is boolean `true`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `={{ $json.valid }}`
- **Input and output connections:**
  - Input from `Validate Schema`
  - True output to `Your workflow logic here`
  - False output to `Return 400 param error`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - If the validator output shape changes or `valid` is missing, strict boolean comparison may not behave as intended
- **Sub-workflow reference:** Consumes result produced by sub-workflow call.

#### 2.3.5 Return 400 param error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an HTTP 400 JSON response when validation fails.
- **Configuration choices:**
  - `respondWith: json`
  - Response code: `400`
  - Response body:
    ```json
    {
      "error": {{ $json.validationError.toJsonString() }},
      "requestSchema": {{ $json.requiredSchema.toJsonString() }}
    }
    ```
- **Key expressions or variables used:**
  - `$json.validationError.toJsonString()`
  - `$json.requiredSchema.toJsonString()`
- **Input and output connections:**
  - Input from false branch of `If Params Valid`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.5`
- **Edge cases or potential failure types:**
  - If `requiredSchema` is not included in the validator response, `toJsonString()` may fail
  - If used outside a webhook flow with `responseNode` mode, it will not behave as intended
- **Sub-workflow reference:** Returns data produced by sub-workflow validation.

#### 2.3.6 Your workflow logic here
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Placeholder for downstream business logic after successful validation.
- **Configuration choices:**
  - No parameters
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from true branch of `If Params Valid`
  - Output to `Return Success Response`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Must be replaced with real logic in actual use
- **Sub-workflow reference:** None directly.

#### 2.3.7 Return Success Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the success HTTP response when validation and downstream logic succeed.
- **Configuration choices:**
  - `respondWith: json`
  - Response body:
    ```json
    {
      "success": true
    }
    ```
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Your workflow logic here`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.5`
- **Edge cases or potential failure types:**
  - If no request reaches this node while webhook is in responseNode mode, requests can hang unless another Respond to Webhook node responds
- **Sub-workflow reference:** None directly.

#### 2.3.8 Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the expected validation error output format.
- **Configuration choices:**
  - Shows a multiline example with 6 validation issues.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** Describes output generated by the sub-workflow.

---

## 2.4 Canvas Documentation

### Overview
These notes explain the validator’s purpose, how to call it, and which nodes are real versus demonstration-only.

### Nodes Involved
- `Sticky Note`
- `Sticky Note5`

### Node Details

#### 2.4.1 Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Main documentation note for users on the canvas.
- **Configuration choices:**
  - Explains:
    - expected inputs `requiredSchema` and `paramsToValidate`
    - output shape
    - example error messages
    - quick-start integration steps
- **Key expressions or variables used:**
  - `={{ your_schema_here }}`
  - `={{ $json.body }}`
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** Describes how to call the sub-workflow.

#### 2.4.2 Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Clarifies that only two nodes are the actual implementation.
- **Configuration choices:**
  - States that the two nodes in the core area are the real functional template and the rest are demonstrations.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** Refers to the validator’s core implementation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Sub-workflow entry point accepting `requiredSchema` and `paramsToValidate` |  | Schema Validation | ### The Actual Workflow\nThese two nodes are the actual functional part of the template, all of the other nodes on the canvas are just for demonstration. |
| Schema Validation | n8n-nodes-base.code | Custom JSON Schema validator with detailed human-readable errors | When Executed by Another Workflow |  | ### The Actual Workflow\nThese two nodes are the actual functional part of the template, all of the other nodes on the canvas are just for demonstration. |
| PLACEHOLDER: Source of your data | n8n-nodes-base.set | Provides sample payload data for the schema-generation/helper example |  | Call 'Param Schema Validation Template' | # Schema Creation\nAIs are great at making schemas. Copy and paste the two nodes below with the entire sticky into your agent of choice with an example of your request.\n\n_Out of view in the sticky is a prompt that will help the agent know how to make the schema_ |
| Call 'Param Schema Validation Template' | n8n-nodes-base.executeWorkflow | Example sub-workflow invocation using placeholder data and a sample schema | PLACEHOLDER: Source of your data |  | # Schema Creation\nAIs are great at making schemas. Copy and paste the two nodes below with the entire sticky into your agent of choice with an example of your request.\n\n_Out of view in the sticky is a prompt that will help the agent know how to make the schema_ |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for how to use the validator |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Introduces the schema creation helper area |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Large prompt template for AI-assisted schema or node generation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Labels the webhook integration example |  |  |  |
| Webhook | n8n-nodes-base.webhook | Example HTTP entry point with downstream response nodes |  | Validate Schema | # Usage Example\nThis is an example showing how to wire the validator into a webhook endpoint |
| Validate Schema | n8n-nodes-base.executeWorkflow | Example call to the validator from a webhook flow | Webhook | If Params Valid | # Usage Example\nThis is an example showing how to wire the validator into a webhook endpoint |
| If Params Valid | n8n-nodes-base.if | Branches based on `$json.valid` from validator output | Validate Schema | Your workflow logic here; Return 400 param error | # Usage Example\nThis is an example showing how to wire the validator into a webhook endpoint |
| Return 400 param error | n8n-nodes-base.respondToWebhook | Returns HTTP 400 with validation error and schema | If Params Valid |  | # Usage Example\nThis is an example showing how to wire the validator into a webhook endpoint |
| Your workflow logic here | n8n-nodes-base.noOp | Placeholder for business logic after validation success | If Params Valid | Return Success Response | # Usage Example\nThis is an example showing how to wire the validator into a webhook endpoint |
| Return Success Response | n8n-nodes-base.respondToWebhook | Returns success JSON response | Your workflow logic here |  | # Usage Example\nThis is an example showing how to wire the validator into a webhook endpoint |
| Sticky Note4 | n8n-nodes-base.stickyNote | Shows example readable validation errors |  |  | Then the `error` field gives a detailed readable description of all of the errors:\n\nValidation failed (6 issues):\n• name: Missing required field \"name\" — Customer full name\n• email: \"not-an-email\" is not valid — expected: Contact email address\n• plan: \"premium\" is not an allowed value. Must be one of: starter, pro, enterprise — Subscription plan\n• seat_count: Expected type \"integer\" but got \"non-integer number\" — Number of licensed seats\n• tags: Must have at least 1 item(s), got 0 — At least one tag for categorization\n• referral_code: Unknown field \"referral_code\" is not allowed |
| Sticky Note5 | n8n-nodes-base.stickyNote | Identifies the two core functional nodes |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

Below is a full rebuild plan.

### A. Build the reusable validation sub-workflow

1. **Create a new workflow**
   - Name it something like: `Param Schema Validation Template`.

2. **Add an Execute Workflow Trigger node**
   - Node type: `When Executed by Another Workflow`
   - Add two workflow inputs:
     1. `requiredSchema` of type `object`
     2. `paramsToValidate` of type `object`

3. **Add a Code node**
   - Name it `Schema Validation`
   - Connect:
     - `When Executed by Another Workflow` → `Schema Validation`

4. **Paste the custom validator code into the Code node**
   - Use JavaScript mode.
   - The code must:
     - read `requiredSchema` from `$input.first().json.requiredSchema`
     - read `paramsToValidate` from `$input.first().json.paramsToValidate`
     - recursively validate supported keywords:
       - `type`
       - `const`
       - `enum`
       - `required`
       - `properties`
       - `additionalProperties`
       - `minLength`
       - `maxLength`
       - `pattern`
       - `minimum`
       - `maximum`
       - `exclusiveMinimum`
       - `exclusiveMaximum`
       - `minItems`
       - `maxItems`
       - `items`
       - `oneOf`
       - `anyOf`
       - `allOf`
       - `not`
     - return:
       - `{ valid: true, validationError: null }` when valid
       - or `{ valid: false, validationError: "...", requiredSchema: <clone> }` when invalid
     - catch unexpected runtime errors and return:
       - `{ valid: false, validationError: "Validator error: ..." }`

5. **Save and activate or keep available for sub-workflow calls**
   - This workflow is now callable from other workflows.

---

### B. Add canvas documentation notes for the validator workflow

6. **Add a Sticky Note**
   - Content should explain:
     - pass `requiredSchema` wrapped in `={{ }}`
     - pass `paramsToValidate`, typically `={{ $json.body }}`
     - outputs `valid`, `validationError`, and optionally `requiredSchema`

7. **Add another Sticky Note near the two functional nodes**
   - Clarify that only the trigger node and code node are functionally required.
   - Mention that the rest of the canvas is demonstration/documentation only.

---

### C. Build the schema-generation helper area

8. **Add a Sticky Note titled for schema creation**
   - Explain that AI can generate schemas from examples.

9. **Add a large Sticky Note with the prompt content**
   - Include:
     - supported schema keywords
     - requirement for `description` on every field
     - rules for using `integer`, `enum`, `oneOf`, `not`, `additionalProperties`, `pattern`, `minItems`
     - reminder to wrap schema in `={{ }}`
     - reminder to double-escape regex backslashes in n8n expression strings
     - reminder to re-select workflow ID after import
     - expected response body for 400 responses
     - instruction to ask users whether they want:
       - full n8n nodes
       - schema only

10. **Add a Set node**
    - Name: `PLACEHOLDER: Source of your data`
    - Create an object field `body`
    - Example value:
      ```json
      {
        "email": "user@example.com",
        "plan": "cloud",
        "seat_count": 25
      }
      ```

11. **Add an Execute Workflow node**
    - Name: `Call 'Param Schema Validation Template'`
    - Connect:
      - `PLACEHOLDER: Source of your data` → `Call 'Param Schema Validation Template'`

12. **Configure the Execute Workflow node**
    - Select the validator workflow you created earlier
    - Use workflow input mapping mode that lets you define inputs below
    - Set:
      - `requiredSchema` to an expression object such as:
        ```javascript
        ={{
          {
            "type": "object",
            "properties": {
              "email": {
                "type": "string",
                "pattern": "^\\\\S+@\\\\S+\\\\.\\\\S+$",
                "description": "Contact email address"
              },
              "plan": {
                "type": "string",
                "enum": ["community", "cloud", "enterprise"],
                "description": "Subscription tier"
              },
              "seat_count": {
                "type": "integer",
                "minimum": 1,
                "maximum": 500,
                "description": "Number of licensed seats"
              }
            },
            "required": ["email", "plan"],
            "additionalProperties": false
          }
        }}
        ```
      - `paramsToValidate` to:
        ```javascript
        ={{ $json.body }}
        ```
    - If available, keep:
      - `attemptToConvertTypes: false`
      - `convertFieldsToString: true`

13. **Important sub-workflow setup note**
    - After importing into a different n8n instance, re-select the target workflow manually because workflow IDs are instance-specific.

---

### D. Build the webhook example integration

14. **Add a Sticky Note for usage example**
   - State that this section demonstrates wiring the validator into a webhook endpoint.

15. **Add a Webhook node**
   - Name: `Webhook`
   - Set a path of your choice
   - Set `responseMode` to `responseNode`

16. **Add an Execute Workflow node**
   - Name: `Validate Schema`
   - Connect:
     - `Webhook` → `Validate Schema`

17. **Configure `Validate Schema`**
   - Select the validator sub-workflow
   - Set workflow inputs:
     - `requiredSchema`:
       ```javascript
       ={{
         {
           "type": "object",
           "properties": {
             "name": {
               "type": "string",
               "minLength": 1,
               "description": "Customer full name"
             },
             "email": {
               "type": "string",
               "pattern": "^\\\\S+@\\\\S+\\\\.\\\\S+$",
               "description": "Contact email address"
             },
             "plan": {
               "type": "string",
               "enum": ["starter", "pro", "enterprise"],
               "description": "Subscription plan"
             },
             "seat_count": {
               "type": "integer",
               "minimum": 1,
               "maximum": 500,
               "description": "Number of licensed seats"
             },
             "tags": {
               "type": "array",
               "minItems": 1,
               "items": {
                 "type": "string",
                 "minLength": 1,
                 "description": "A tag label"
               },
               "description": "At least one tag for categorization"
             }
           },
           "required": ["name", "email", "plan", "seat_count"],
           "additionalProperties": false
         }
       }}
       ```
     - `paramsToValidate`:
       ```javascript
       ={{ $json.body }}
       ```

18. **Add an If node**
   - Name: `If Params Valid`
   - Connect:
     - `Validate Schema` → `If Params Valid`
   - Configure a boolean condition:
     - left value: `={{ $json.valid }}`
     - operation: `true`
   - Use strict type validation if available.

19. **Add a Respond to Webhook node for errors**
   - Name: `Return 400 param error`
   - Connect from the **false** output of `If Params Valid`
   - Configure:
     - respond with: `json`
     - response code: `400`
     - response body:
       ```javascript
       ={
         "error": {{ $json.validationError.toJsonString() }},
         "requestSchema": {{ $json.requiredSchema.toJsonString() }}
       }
       ```

20. **Add a placeholder logic node**
   - Node type: `No Operation`
   - Name: `Your workflow logic here`
   - Connect from the **true** output of `If Params Valid`

21. **Add a success response node**
   - Node type: `Respond to Webhook`
   - Name: `Return Success Response`
   - Connect:
     - `Your workflow logic here` → `Return Success Response`
   - Configure:
     - respond with: `json`
     - body:
       ```javascript
       ={
         "success": true
       }
       ```

22. **Add a Sticky Note with sample error output**
   - Include a multiline example of missing fields, invalid enum, invalid email, wrong integer type, empty tags, and unknown fields.

---

### E. Credential configuration

23. **Credentials**
   - This workflow does **not** require external service credentials.
   - No OpenAI, OAuth2, database, or API credentials are needed.
   - The only dependency is that the calling Execute Workflow nodes must be able to reference the local validator workflow.

---

### F. Expected input/output contract for the sub-workflow

24. **Sub-workflow input contract**
   - `requiredSchema`: object
   - `paramsToValidate`: object

25. **Sub-workflow success output**
   ```json
   {
     "valid": true,
     "validationError": null
   }
   ```

26. **Sub-workflow failure output**
   ```json
   {
     "valid": false,
     "validationError": "Validation failed (...): ...",
     "requiredSchema": { ... }
   }
   ```

27. **Operational constraints**
   - `requiredSchema` must be passed as an object expression, not as plain text
   - regex patterns must be escaped correctly for n8n expressions
   - if `additionalProperties` is false, undeclared keys are rejected
   - every field should ideally have `description` because the validator includes descriptions in human-facing errors

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is a reusable JSON Schema validator sub-workflow for n8n and the rest of the canvas is mostly documentation and examples. | Overall workflow design |
| Validation errors are intentionally human-readable and suitable for returning to callers or feeding back into LLM-based systems for self-correction. | Core behavior |
| The validator supports: `type`, `const`, `enum`, `required`, `properties`, `additionalProperties`, `minLength`, `maxLength`, `pattern`, `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, `minItems`, `maxItems`, `items`, `oneOf`, `anyOf`, `allOf`, `not`. | Supported schema features |
| Every schema field should include `description` because descriptions are incorporated into returned validation messages. | Schema authoring guidance |
| When embedding a schema into an Execute Workflow node, wrap it in `={{ }}` so n8n evaluates it as an object rather than a string. | n8n expression requirement |
| Regex backslashes in `pattern` values must be double-escaped when embedded inside n8n expressions. | n8n expression/string escaping |
| After importing to another n8n instance, re-select the called workflow in Execute Workflow nodes because workflow IDs are environment-specific. | Portability note |
| Suggested request-body path for validation is `={{ $json.body }}`, but it must be adapted to the actual upstream data shape. | Integration note |
| The error response format used in the example returns both the readable error string and the schema violated, which is useful for machine callers. | Error-handling design |