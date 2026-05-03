Retrieve a LinkedIn contact’s name from a profile URL with LinkUp

https://n8nworkflows.xyz/workflows/retrieve-a-linkedin-contact-s-name-from-a-profile-url-with-linkup-13974


# Retrieve a LinkedIn contact’s name from a profile URL with LinkUp

# 1. Workflow Overview

This workflow retrieves a contact’s name from a LinkedIn profile URL by querying the LinkUp AI Search API, then writes the returned name back into an n8n Data Table. It is designed for lightweight enrichment of LinkedIn profile records without relying on scraping or dedicated LinkedIn APIs.

Typical use cases:
- Filling missing contact names when only a LinkedIn profile URL is known
- Enriching lead lists stored in an n8n Data Table
- Running small batch lookups with a configurable cap per execution

## 1.1 Manual Start and Local Settings
The workflow starts manually and initializes a local configuration value: the LinkUp API key.

## 1.2 Data Retrieval and Batch Control
It loads rows from a Data Table named `LinkedInURL`, limits processing to 30 items, then iterates through them one by one.

## 1.3 Conditional Filtering
For each record, it checks whether:
- the `url` field is present
- the `name` field is empty

Only rows matching both conditions are sent for enrichment.

## 1.4 LinkUp Name Resolution
The workflow sends the LinkedIn profile URL to LinkUp using an HTTP POST request with a natural-language prompt asking for the associated name.

## 1.5 Data Table Update and Loop Continuation
If LinkUp returns a response, the workflow updates the `name` column in the Data Table row, then continues to the next batch item. If the row does not need processing, it also continues cleanly.

---

# 2. Block-by-Block Analysis

## Block 1 — Manual Trigger and API Key Setup

### Overview
This block starts the workflow manually and defines the LinkUp API key in a Set node. The API key is later injected into the HTTP Authorization header.

### Nodes Involved
- `When clicking ‘Execute workflow’`
- `settings`

### Node Details

#### 1. `When clicking ‘Execute workflow’`
- **Type and role:** `manualTrigger`; entry point for manual executions from the n8n editor.
- **Configuration choices:** No custom parameters; this is the standard manual start node.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input
  - Output → `settings`
- **Version-specific requirements:** Type version `1`; standard.
- **Edge cases or potential failure types:**
  - No runtime logic issues
  - Only available for manual execution, not event-driven automation
- **Sub-workflow reference:** None

#### 2. `settings`
- **Type and role:** `set`; stores workflow-local configuration.
- **Configuration choices:**
  - Assigns a string field named `ApiKey`
  - Placeholder value: `enter your LinkUp API key here`
- **Key expressions or variables used:**
  - Produces `{{$json.ApiKey}}` for downstream usage
- **Input and output connections:**
  - Input ← `When clicking ‘Execute workflow’`
  - Output → `linkedinurl`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If the placeholder is not replaced with a real API key, the LinkUp API call will fail with authentication/authorization errors
  - Because this is plain text inside the workflow, it is less secure than using n8n credentials or environment variables
- **Sub-workflow reference:** None

---

## Block 2 — Data Table Read, Limit, and Iteration

### Overview
This block reads records from the `LinkedInURL` Data Table, restricts the run to a maximum of 30 items, and iterates over records one at a time. This structure reduces load and makes updates easier to manage.

### Nodes Involved
- `linkedinurl`
- `Limit`
- `Loop Over Items`

### Node Details

#### 3. `linkedinurl`
- **Type and role:** `dataTable`; reads rows from an n8n Data Table.
- **Configuration choices:**
  - Operation: `get`
  - Data Table: `LinkedInURL`
- **Key expressions or variables used:**
  - Downstream references use fields like:
    - `$('linkedinurl').item.json.url`
    - `$('linkedinurl').item.json.name`
    - `$('linkedinurl').item.json.id`
- **Input and output connections:**
  - Input ← `settings`
  - Output → `Limit`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Failure if the referenced Data Table does not exist in the project
  - Failure if the table schema differs from expected fields (`url`, `name`)
  - Permission or environment issues if Data Tables are unavailable in the current n8n setup
- **Sub-workflow reference:** None

#### 4. `Limit`
- **Type and role:** `limit`; caps the number of items processed in a single execution.
- **Configuration choices:**
  - `maxItems = 30`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `linkedinurl`
  - Output → `Loop Over Items`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - No major failures expected
  - Records beyond the first 30 are ignored for that execution
- **Sub-workflow reference:** None

#### 5. `Loop Over Items`
- **Type and role:** `splitInBatches`; iterates through items, effectively processing records one at a time.
- **Configuration choices:**
  - No batch options explicitly set
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Input ← `Limit`
  - Batch output (second branch in this workflow) → `If`
  - Loop continuation input receives flow back from `Replace Me`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Incorrect loop-back wiring would stop iteration prematurely
  - If downstream nodes fail without error handling, the loop stops
- **Sub-workflow reference:** None

---

## Block 3 — Conditional Filtering of Eligible Rows

### Overview
This block decides whether a Data Table row should be processed. It only sends records to LinkUp if the LinkedIn URL exists and the name field is still empty.

### Nodes Involved
- `If`

### Node Details

#### 6. `If`
- **Type and role:** `if`; conditional gate for enrichment logic.
- **Configuration choices:**
  - Uses an `and` combinator
  - Condition 1: `url` is not empty
  - Condition 2: `name` is empty
- **Key expressions or variables used:**
  - `={{ $('linkedinurl').item.json.url }}`
  - `={{ $('linkedinurl').item.json.name }}`
- **Input and output connections:**
  - Input ← `Loop Over Items`
  - True output → `linkup`
  - False output → `Replace Me`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - If the Data Table schema changes and `url` or `name` is absent, expression evaluation may behave unexpectedly
  - Strict type validation is enabled in the condition options, so malformed field values may affect evaluation
- **Sub-workflow reference:** None

---

## Block 4 — LinkUp API Call for Name Resolution

### Overview
This block sends the LinkedIn profile URL to LinkUp’s search API with a prompt asking for the associated person’s name. It is the core enrichment step of the workflow.

### Nodes Involved
- `linkup`

### Node Details

#### 7. `linkup`
- **Type and role:** `httpRequest`; performs the API call to LinkUp.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.linkup.so/v1/search`
  - Sends headers and JSON body
  - Header:
    - `Authorization: Bearer <ApiKey>`
  - Body parameters:
    - `q`: natural-language prompt asking LinkUp to find the name associated with the LinkedIn profile URL
    - `depth`: `standard`
    - `outputType`: `sourcedAnswer`
    - `includeImages`: `false`
    - `includeInlineCitations`: `false`
- **Key expressions or variables used:**
  - Prompt includes:
    - `{{ $('linkedinurl').item.json.url }}`
  - Authorization header:
    - `={{ "Bearer " + $('settings').item.json.ApiKey }}`
- **Input and output connections:**
  - Input ← `If` (true branch)
  - Output → `update name`
- **Version-specific requirements:** Type version `4.3`
- **Edge cases or potential failure types:**
  - Invalid or missing API key → 401/403-style auth failures
  - Network timeout or upstream API outage
  - Rate limiting from LinkUp
  - Unexpected response structure; the next node expects an `answer` field
  - Prompt quality may affect accuracy
- **Sub-workflow reference:** None

**Important note about the node comment:**  
The node-level note says:  
> return only the linkedin url of {{ $json.name }}. If you have multiple answers, pick the one who works in data or software and located in New York. Return an empty string if there is no match

This comment does **not** match the actual body prompt configured in the node. The real request asks LinkUp to find the **name associated with a LinkedIn profile URL**. The note appears to be leftover text from another use case and should not be relied on.

---

## Block 5 — Data Table Update and Loop Continuation

### Overview
This block writes the returned name into the matching Data Table row, then returns control to the loop so the next item can be processed. It also handles the skipped branch using the same continuation path.

### Nodes Involved
- `update name`
- `Replace Me`

### Node Details

#### 8. `update name`
- **Type and role:** `dataTable`; updates the current row in the `LinkedInURL` Data Table.
- **Configuration choices:**
  - Operation: `update`
  - Data Table: `LinkedInURL`
  - Writes the `name` column using the value from `{{$json.answer}}`
  - The row filter targets the current item by ID from the original Data Table item
- **Key expressions or variables used:**
  - Value written:
    - `={{ $json.answer }}`
  - Filter condition:
    - `={{ $('linkedinurl').item.json.id }}`
- **Input and output connections:**
  - Input ← `linkup`
  - Output → `Replace Me`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - If LinkUp does not return an `answer` field, the `name` may be set to empty or fail depending on runtime behavior
  - If the Data Table row ID is not available or mismatched, the update may not affect any record
  - Schema mismatch on the `name` column can break the update
- **Sub-workflow reference:** None

#### 9. `Replace Me`
- **Type and role:** `noOp`; placeholder pass-through node used to unify both branches before looping again.
- **Configuration choices:** None
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs ← `If` (false branch), `update name`
  - Output → `Loop Over Items`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - No operational logic; mainly structural
  - Can be renamed to something clearer such as `Continue Loop`
- **Sub-workflow reference:** None

---

## Block 6 — Documentation Sticky Notes

### Overview
These nodes are visual annotations only. They do not execute but provide setup guidance, usage instructions, and context.

### Nodes Involved
- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`

### Node Details

#### 10. `Sticky Note`
- **Type and role:** `stickyNote`; visual instruction
- **Configuration choices:** Content says API key must be configured
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 11. `Sticky Note1`
- **Type and role:** `stickyNote`; visual reminder
- **Configuration choices:** Content reminds the user to add LinkedIn URLs
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 12. `Sticky Note2`
- **Type and role:** `stickyNote`; visual label
- **Configuration choices:** Content identifies the LinkUp API call area
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 13. `Sticky Note3`
- **Type and role:** `stickyNote`; full workflow documentation embedded in canvas
- **Configuration choices:** Contains summary, usage instructions, requirements, and external links
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start of the workflow |  | settings |  |
| settings | Set | Stores the LinkUp API key for downstream use | When clicking ‘Execute workflow’ | linkedinurl | Setup Your Api Key here |
| linkedinurl | Data Table | Reads rows from the `LinkedInURL` Data Table | settings | Limit | Don't forget to add LinkedIn URL |
| Limit | Limit | Restricts execution to 30 rows | linkedinurl | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterates through the retrieved rows | Limit, Replace Me | If |  |
| If | If | Checks whether `url` exists and `name` is empty | Loop Over Items | linkup, Replace Me |  |
| linkup | HTTP Request | Calls LinkUp API to resolve the name from the LinkedIn URL | If | update name | LinkUp Call. |
| update name | Data Table | Updates the `name` field in the Data Table | linkup | Replace Me |  |
| Replace Me | No Operation | Pass-through node used to continue the loop | If, update name | Loop Over Items |  |
| Sticky Note | Sticky Note | Visual annotation for API key setup |  |  |  |
| Sticky Note1 | Sticky Note | Visual annotation reminding users to add URLs |  |  |  |
| Sticky Note2 | Sticky Note | Visual annotation labeling the LinkUp section |  |  |  |
| Sticky Note3 | Sticky Note | Embedded documentation and external references |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Set the workflow name to something like: `Retrieve a LinkedIn contact’s name from a profile URL with LinkUp`.

2. **Add the manual trigger**
   - Add a **Manual Trigger** node.
   - Keep the default configuration.
   - This will be the workflow entry point.

3. **Add a Set node for configuration**
   - Add a **Set** node after the Manual Trigger.
   - Name it `settings`.
   - Create one string field:
     - `ApiKey` = `enter your LinkUp API key here`
   - In production, replace this placeholder with a real LinkUp API key.
   - Connect:
     - `Manual Trigger` → `settings`

4. **Prepare the Data Table**
   - In n8n, create a Data Table named `LinkedInURL`.
   - Add at least these columns:
     - `url` as String
     - `name` as String
   - Add one or more records with LinkedIn profile URLs in `url`.
   - Leave `name` empty for rows you want the workflow to fill.
   - The workflow assumes each record also has the internal Data Table row identifier used by n8n.

5. **Add a Data Table node to fetch records**
   - Add a **Data Table** node after `settings`.
   - Name it `linkedinurl`.
   - Operation: `Get`
   - Select the Data Table: `LinkedInURL`
   - Connect:
     - `settings` → `linkedinurl`

6. **Add a Limit node**
   - Add a **Limit** node after `linkedinurl`.
   - Set **Max Items** to `30`.
   - This keeps each execution bounded.
   - Connect:
     - `linkedinurl` → `Limit`

7. **Add a loop node**
   - Add a **Loop Over Items** node, implemented with **Split In Batches**.
   - Name it `Loop Over Items`.
   - Keep default settings unless you want a different batch strategy.
   - Connect:
     - `Limit` → `Loop Over Items`

8. **Add an If node for eligibility**
   - Add an **If** node after the loop’s item-processing output.
   - Name it `If`.
   - Configure it with **AND** logic and two conditions:
     1. `url` is **not empty**
     2. `name` is **empty**
   - Use expressions referencing the Data Table item:
     - Left value for URL check:
       - `{{ $('linkedinurl').item.json.url }}`
     - Left value for name check:
       - `{{ $('linkedinurl').item.json.name }}`
   - Connect:
     - `Loop Over Items` → `If`

9. **Add the LinkUp HTTP Request node**
   - Add an **HTTP Request** node.
   - Name it `linkup`.
   - Connect the **true** output of `If` to `linkup`.
   - Configure:
     - Method: `POST`
     - URL: `https://api.linkup.so/v1/search`
     - Enable sending headers
     - Enable sending body
   - Add header:
     - `Authorization` = `{{ "Bearer " + $('settings').item.json.ApiKey }}`
   - Add body parameters:
     - `q` =  
       `You are an expert LinkedIn researcher. Your goal is to find the name associated to this LinkedIn profile URL {{ $('linkedinurl').item.json.url }}. Return only the name.`
     - `depth` = `standard`
     - `outputType` = `sourcedAnswer`
     - `includeImages` = `false`
     - `includeInlineCitations` = `false`
   - No separate credential object is used in this workflow; authentication is handled through the Authorization header.
   - If preferred, you may adapt this to use a more secure credential strategy outside the workflow JSON.

10. **Add a Data Table update node**
    - Add another **Data Table** node after `linkup`.
    - Name it `update name`.
    - Operation: `Update`
    - Select the same Data Table: `LinkedInURL`
    - Configure the update mapping so the `name` column receives:
      - `{{ $json.answer }}`
    - Configure the update filter to target the current original row using the Data Table ID:
      - Use `{{ $('linkedinurl').item.json.id }}`
    - Connect:
      - `linkup` → `update name`

11. **Add a No Operation node**
    - Add a **No Operation, do nothing** node.
    - Name it `Replace Me`.
    - This node is only used to merge both branches before the next iteration.
    - Connect:
      - `update name` → `Replace Me`
      - `If` false output → `Replace Me`

12. **Close the loop**
    - Connect:
      - `Replace Me` → `Loop Over Items`
   - This connection returns control to the loop so it can process the next item.

13. **Optional: add visual notes**
   - Add sticky notes to match the original layout if desired:
     - Near `settings`: `Setup Your Api Key here`
     - Near `linkedinurl`: `Don't forget to add LinkedIn URL`
     - Near `linkup`: `LinkUp Call.`
   - You can also add a larger sticky note with usage guidance and links.

14. **Test with sample data**
   - Insert a test record in `LinkedInURL`, such as a valid public LinkedIn profile URL.
   - Execute the workflow manually.
   - Confirm that:
     - only rows with missing `name` are processed
     - LinkUp returns an `answer`
     - the `name` column is updated correctly

15. **Validate behavior and harden if needed**
   - Consider adding:
     - retry logic for HTTP failures
     - error handling for empty LinkUp responses
     - a check that `answer` exists before updating
     - rate limiting or smaller batches if processing large lists

## Expected Input/Output Behavior
- **Input source:** Data Table rows from `LinkedInURL`
- **Required input fields:**
  - `url` should contain a LinkedIn profile URL
  - `name` should be empty to trigger enrichment
- **Expected LinkUp output:** a response containing an `answer` field with the person’s name
- **Final output:** the same Data Table row updated with `name = answer`

## Credential Configuration
- **LinkUp**
  - No dedicated n8n credential node is used here
  - The workflow stores the API key in the `settings` node and passes it in:
    - `Authorization: Bearer <your_api_key>`
  - Recommended improvement: move the API key into a secure credential or environment-backed mechanism

## Version / Compatibility Notes
- The embedded note says the workflow was tested on **n8n 1.122.4 (Ubuntu)**.
- The workflow uses:
  - Data Table nodes
  - Split In Batches / Loop Over Items behavior
  - HTTP Request node version `4.3`
- Reproduction may require an n8n version that supports Data Tables in the same manner.

## Sub-workflow Setup
- This workflow does **not** use any sub-workflows.
- It has a single explicit entry point:
  - `When clicking ‘Execute workflow’`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Recovering a Contact’s Name from a LinkedIn URL with LinkUp | Embedded workflow documentation |
| This workflow retrieves a LinkedIn contact’s full name from their profile URL using the LinkUp AI Search API. It’s positioned as a lightweight alternative to web scraping or complex APIs and integrates with n8n automations. | Embedded workflow documentation |
| Create a Data Table named `LinkedInURL` with two fields: `url` (String) and `name` (String). | Embedded workflow documentation |
| Example LinkedIn URL: `https://www.linkedin.com/in/stephaneheckel/` | Embedded workflow documentation |
| Get a LinkUp API key | https://app.linkup.so/api-keys |
| LinkUp website | https://www.linkup.so |
| Comment on the related post for help | https://www.linkedin.com/posts/n8n-about_linkedin-activity-7436922917336289281-hTxv/ |
| Contact the author on LinkedIn | https://www.linkedin.com/in/stephaneheckel/ |
| Ask in the n8n Forum | https://community.n8n.io/ |
| Requirements: LinkUp account; tested on n8n 1.122.4 (Ubuntu) | Embedded workflow documentation |