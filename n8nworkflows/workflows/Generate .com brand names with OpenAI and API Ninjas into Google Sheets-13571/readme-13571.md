Generate .com brand names with OpenAI and API Ninjas into Google Sheets

https://n8nworkflows.xyz/workflows/generate--com-brand-names-with-openai-and-api-ninjas-into-google-sheets-13571


# Generate .com brand names with OpenAI and API Ninjas into Google Sheets

# 1. Workflow Overview

This workflow generates candidate **.com brand names**, checks whether those domains are available, and stores the results in Google Sheets. It uses **OpenAI** to create short brandable names, **API Ninjas** to test domain availability, and two Google Sheets tabs to persist both available and unavailable results across runs.

Its main use case is iterative brand/domain research: each run avoids previously checked names, keeps looping until it finds enough available domains, and writes structured results for later review.

## 1.1 Entry and Existing State Loading

The workflow starts manually, then reads two Google Sheets tabs:

- `available`
- `closed`

These tabs act as persistent memory. Their contents are used to determine:

- how many available domains were already found
- which domains have already been checked and must not be suggested again

## 1.2 Prompt Construction and AI Name Generation

Using the current state, the workflow builds a prompt for OpenAI that:

- asks for exactly 20 short English brand names
- excludes already checked names/domains
- requests JSON-only output

The generated names are then normalized into `.com` domains.

## 1.3 Domain Availability Checking

Each `.com` domain is sent to API Ninjas. The workflow then merges the original generated name/domain with the API response and keeps only responses that contain a valid boolean `available` flag.

## 1.4 Result Routing, Persistence, and Loop Control

The checked domains are split into:

- available domains → appended to the `available` sheet
- unavailable domains → appended to the `closed` sheet

At the same time, the workflow updates its loop state:

- cumulative available count
- cumulative already-checked domains
- iteration number

It exits when either:

- at least 15 available domains have been found, or
- more than 10 iterations have run

Otherwise, it loops back and generates a new batch.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Entry and Sheet State Initialization

### Overview

This block starts the workflow and reconstructs prior search state from Google Sheets. It loads both tabs and creates a reusable state object containing previously checked domains, total available count, and initial iteration number.

### Nodes Involved

- Manual Trigger
- Read sheet available
- Trigger closed read
- Read sheet closed
- Build initial state

### Node Details

#### 1. Manual Trigger

- **Type and role:** `n8n-nodes-base.manualTrigger`; manual entry point for on-demand execution.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none
  - Output: `Read sheet available`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None beyond standard manual execution behavior.
- **Sub-workflow reference:** None.

#### 2. Read sheet available

- **Type and role:** `n8n-nodes-base.googleSheets`; reads existing rows from the `available` tab.
- **Configuration choices:**
  - Google Sheets document is set by ID
  - Sheet name is `available`
  - No extra options enabled
  - `continueOnFail: true`, so the workflow can proceed even if the sheet read fails
- **Key expressions or variables used:** Static sheet name and document ID placeholder `YOUR_GOOGLE_SHEET_ID`.
- **Input and output connections:**  
  - Input: `Manual Trigger`
  - Output: `Trigger closed read`
- **Version-specific requirements:** Type version `4.5`; requires Google Sheets credentials compatible with this node version.
- **Edge cases or failure types:**
  - OAuth/authentication failure
  - wrong spreadsheet ID
  - missing `available` sheet tab
  - permission denied
  - empty sheet
  - because `continueOnFail` is enabled, downstream logic may continue with an empty dataset
- **Sub-workflow reference:** None.

#### 3. Trigger closed read

- **Type and role:** `n8n-nodes-base.code`; emits a dummy item so the second sheet can be read in sequence.
- **Configuration choices:** Returns a single item with empty JSON.
- **Key expressions or variables used:**  
  - `return [{ json: {} }];`
- **Input and output connections:**  
  - Input: `Read sheet available`
  - Output: `Read sheet closed`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:** Minimal; only JavaScript syntax/runtime issues if edited incorrectly.
- **Sub-workflow reference:** None.

#### 4. Read sheet closed

- **Type and role:** `n8n-nodes-base.googleSheets`; reads existing rows from the `closed` tab.
- **Configuration choices:**
  - Google Sheets document by ID
  - Sheet name `closed`
  - `continueOnFail: true`
- **Key expressions or variables used:** Static tab name and `YOUR_GOOGLE_SHEET_ID`.
- **Input and output connections:**  
  - Input: `Trigger closed read`
  - Output: `Build initial state`
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or failure types:**
  - missing `closed` tab
  - credential failure
  - spreadsheet access issues
  - empty sheet
  - downstream block still continues because of `continueOnFail`
- **Sub-workflow reference:** None.

#### 5. Build initial state

- **Type and role:** `n8n-nodes-base.code`; builds the initial loop state from both sheet reads.
- **Configuration choices:**
  - Pulls all rows from `Read sheet available` and `Read sheet closed`
  - Safely catches read errors
  - Extracts domain-like fields from possible column names:
    - `domain`
    - `name`
    - `Domain`
    - `Name`
  - Builds:
    - `totalFound`
    - `alreadyChecked`
    - `iteration: 0`
- **Key expressions or variables used:**
  - `$('Read sheet available').all()`
  - `$('Read sheet closed').all()`
  - deduplication with `new Set(...)`
- **Input and output connections:**  
  - Input: `Read sheet closed`
  - Output: `Prepare prompt`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - if sheets return malformed rows, extraction may miss domains
  - if wrong columns are used, `alreadyChecked` may be incomplete
  - if sheet values contain names instead of domains, exclusion may become mixed
- **Sub-workflow reference:** None.

---

## 2.2 Block: Prompt Preparation and OpenAI Name Generation

### Overview

This block prepares exclusion-aware prompts and requests 20 new brand names from OpenAI. It is also the loop re-entry point after each iteration.

### Nodes Involved

- Prepare prompt
- Generate names (OpenAI)

### Node Details

#### 6. Prepare prompt

- **Type and role:** `n8n-nodes-base.code`; composes system and user prompts from workflow state.
- **Configuration choices:**
  - Reads `totalFound`, `alreadyChecked`, and `iteration`
  - If `alreadyChecked` is non-empty, injects an exclusion sentence into both prompts
  - Requests:
    - short, distinctive, memorable English names
    - letters only
    - no numbers
    - no hyphens
    - under 14 characters
    - exactly 20 names
    - JSON with a single `names` array
- **Key expressions or variables used:**
  - `$input.first().json`
  - `excludeText`
  - outputs `systemPrompt`, `userPrompt`, `totalFound`, `alreadyChecked`, `iteration`
- **Input and output connections:**  
  - Inputs:
    - `Build initial state`
    - `Exit or loop` (loop-back on false branch)
  - Output: `Generate names (OpenAI)`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - very large `alreadyChecked` list can make prompts long
  - prompt may still yield duplicate or invalid names despite instructions
  - if upstream state is malformed, defaults are used
- **Sub-workflow reference:** None.

#### 7. Generate names (OpenAI)

- **Type and role:** `n8n-nodes-base.httpRequest`; sends a chat completion request to OpenAI.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.openai.com/v1/chat/completions`
  - Authentication: generic header auth
  - Adds `Content-Type: application/json`
  - Sends JSON body with:
    - `model: gpt-4o`
    - `temperature: 0.7`
    - `max_tokens: 2000`
    - system and user messages from previous node
    - `response_format: { type: 'json_object' }`
- **Key expressions or variables used:**
  - `$json.systemPrompt`
  - `$json.userPrompt`
  - `={{ JSON.stringify(...) }}`
- **Input and output connections:**  
  - Input: `Prepare prompt`
  - Output: `Normalize to domains`
- **Version-specific requirements:** Type version `4`; requires HTTP Header Auth credential containing OpenAI API key, typically `Authorization: Bearer ...`.
- **Edge cases or failure types:**
  - invalid/missing OpenAI API key
  - model access not enabled
  - rate limits
  - non-JSON or unexpected API response
  - token/context limits if exclusion list grows too much
  - networking issues/timeouts
- **Sub-workflow reference:** None.

---

## 2.3 Block: Name Normalization and Domain Lookup

### Overview

This block parses the OpenAI response, sanitizes candidate names, converts them to `.com` domains, and checks domain availability through API Ninjas.

### Nodes Involved

- Normalize to domains
- Check domain (API Ninjas)

### Node Details

#### 8. Normalize to domains

- **Type and role:** `n8n-nodes-base.code`; parses OpenAI output and emits one item per cleaned brand/domain pair.
- **Configuration choices:**
  - Validates the presence of `choices[0].message.content`
  - Parses the JSON returned by OpenAI
  - Reads `parsed.names`
  - Cleans each name by:
    - trimming whitespace
    - removing spaces
    - removing all non-letters
  - Converts each result to lowercase `.com`
- **Key expressions or variables used:**
  - `openaiResponse.choices[0].message.content`
  - `JSON.parse(...)`
  - regex cleanup:
    - `/\s+/g`
    - `/[^a-zA-Z]/g`
- **Input and output connections:**  
  - Input: `Generate names (OpenAI)`
  - Output: `Check domain (API Ninjas)`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - OpenAI returns malformed JSON
  - `names` key missing
  - empty array returned
  - names collapse to empty strings after cleanup
  - output count may be less than 20 after filtering
- **Sub-workflow reference:** None.

#### 9. Check domain (API Ninjas)

- **Type and role:** `n8n-nodes-base.httpRequest`; checks each generated domain against API Ninjas.
- **Configuration choices:**
  - URL: `https://api.api-ninjas.com/v1/domain`
  - Sends query parameter `domain={{ $json.domain }}`
  - Authentication: generic header auth
  - Response option `neverError: true` so non-2xx responses do not hard-fail the node
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - `={{ $json.domain }}`
- **Input and output connections:**  
  - Input: `Normalize to domains`
  - Output: `Merge name and API result`
- **Version-specific requirements:** Type version `4.2`; requires HTTP Header Auth credential containing API Ninjas key, usually `X-Api-Key`.
- **Edge cases or failure types:**
  - invalid/missing API Ninjas key
  - rate limits
  - unsupported/malformed domain
  - API returns error payload without `available`
  - because `neverError` and `continueOnFail` are enabled, failures may silently pass through as non-usable items
- **Sub-workflow reference:** None.

---

## 2.4 Block: Result Consolidation, Routing, and State Update

### Overview

This block merges original name/domain data with API results, splits available and unavailable domains, writes them to the appropriate sheets, and computes whether to continue looping.

### Nodes Involved

- Merge name and API result
- Filter available only
- Closed only
- Append to sheet available
- Append to sheet closed
- State for next run
- Exit or loop

### Node Details

#### 10. Merge name and API result

- **Type and role:** `n8n-nodes-base.code`; combines normalized names with domain check results and attaches round state.
- **Configuration choices:**
  - Reads all API result items
  - Reads all items from `Normalize to domains`
  - Reads the latest state from `Prepare prompt`
  - Keeps only items where `apiJson.available` is a boolean
  - Produces merged records with:
    - `name`
    - `domain`
    - API fields
    - `_roundState`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$('Normalize to domains').all()`
  - `$('Prepare prompt').last().json`
  - `typeof apiJson.available === 'boolean'`
- **Input and output connections:**  
  - Input: `Check domain (API Ninjas)`
  - Outputs:
    - `Filter available only`
    - `State for next run`
    - `Closed only`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - assumes API result order matches normalization order
  - if API response items are missing or reordered, name/domain alignment may be wrong
  - items without boolean `available` are dropped entirely
  - if all API calls fail, downstream state may update from an empty set
- **Sub-workflow reference:** None.

#### 11. Filter available only

- **Type and role:** `n8n-nodes-base.filter`; routes only available domains.
- **Configuration choices:**
  - Condition: `$json.available === true`
- **Key expressions or variables used:**
  - `={{ $json.available }}`
- **Input and output connections:**  
  - Input: `Merge name and API result`
  - Output: `Append to sheet available`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - if `available` is missing or non-boolean, item is excluded
- **Sub-workflow reference:** None.

#### 12. Closed only

- **Type and role:** `n8n-nodes-base.code`; keeps only unavailable domains and reshapes output for Sheets.
- **Configuration choices:**
  - Filters input items where `available === false`
  - Outputs only `domain` and `name`
- **Key expressions or variables used:**
  - `$input.all()`
  - `i.json.available === false`
- **Input and output connections:**  
  - Input: `Merge name and API result`
  - Output: `Append to sheet closed`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - items without `available` are ignored
  - malformed items may produce empty output
- **Sub-workflow reference:** None.

#### 13. Append to sheet available

- **Type and role:** `n8n-nodes-base.googleSheets`; appends available domain rows to the `available` tab.
- **Configuration choices:**
  - Operation: `append`
  - Sheet name: `available`
  - Auto-maps incoming fields to columns
  - Expected schema includes:
    - `domain`
    - `name`
- **Key expressions or variables used:** Input auto-mapping.
- **Input and output connections:**  
  - Input: `Filter available only`
  - Output: none
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or failure types:**
  - Google auth issues
  - missing columns/header mismatch
  - wrong spreadsheet ID
  - duplicate rows are not prevented
- **Sub-workflow reference:** None.

#### 14. Append to sheet closed

- **Type and role:** `n8n-nodes-base.googleSheets`; appends unavailable domain rows to the `closed` tab.
- **Configuration choices:**
  - Operation: `append`
  - Sheet name: `closed`
  - Auto-map fields `domain` and `name`
- **Key expressions or variables used:** Input auto-mapping.
- **Input and output connections:**  
  - Input: `Closed only`
  - Output: none
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or failure types:**
  - same Google Sheets issues as above
  - duplicate closed domains may accumulate
- **Sub-workflow reference:** None.

#### 15. State for next run

- **Type and role:** `n8n-nodes-base.code`; updates loop state after one generation/check round.
- **Configuration choices:**
  - Reads merged items
  - Gets previous `_roundState` from the first item, else uses defaults
  - Counts current round available domains
  - Adds this round’s checked domains to `alreadyChecked`
  - Increments iteration
  - Increments cumulative found count
- **Key expressions or variables used:**
  - `$input.all()`
  - `first._roundState`
  - `mergeItems.filter(i => i.json.available === true).length`
  - dedupe with `new Set(...)`
- **Input and output connections:**  
  - Input: `Merge name and API result`
  - Output: `Exit or loop`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - if no merged items exist, defaults are used and iteration resets from default path logic
  - if first item lacks `_roundState`, previous progress may be lost for that cycle
  - domains from failed API calls are not added to checked list
- **Sub-workflow reference:** None.

#### 16. Exit or loop

- **Type and role:** `n8n-nodes-base.if`; decides whether to stop or generate another batch.
- **Configuration choices:**
  - OR logic with two boolean conditions:
    - `totalFound >= 15`
    - `iteration > 10`
  - True branch exits
  - False branch loops back to `Prepare prompt`
- **Key expressions or variables used:**
  - `={{ $json.totalFound >= 15 }}`
  - `={{ $json.iteration > 10 }}`
- **Input and output connections:**  
  - Input: `State for next run`
  - Outputs:
    - true branch: none
    - false branch: `Prepare prompt`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - if state is malformed, comparisons may not behave as expected
  - because branch 0 is empty, workflow simply ends when exit conditions are met
- **Sub-workflow reference:** None.

---

## 2.5 Block: Documentation and Embedded Notes

### Overview

These nodes are non-executable annotations embedded in the canvas. They explain setup, logic, and sheet-writing behavior.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### 17. Sticky Note

- **Type and role:** `n8n-nodes-base.stickyNote`; canvas note for the sheet-writing area.
- **Configuration choices:** Text explains that available domains go to `available` and taken ones to `closed`.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### 18. Sticky Note1

- **Type and role:** `n8n-nodes-base.stickyNote`; high-level flow explanation.
- **Configuration choices:** Describes the whole workflow and loop logic.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### 19. Sticky Note2

- **Type and role:** `n8n-nodes-base.stickyNote`; setup and credential instructions.
- **Configuration choices:** Explains credentials, required sheet tabs, and prompt customization.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### 20. Sticky Note3

- **Type and role:** `n8n-nodes-base.stickyNote`; annotation for the start/load-state block.
- **Configuration choices:** Describes loading sheets and building loop state.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### 21. Sticky Note4

- **Type and role:** `n8n-nodes-base.stickyNote`; annotation for prompting and normalization.
- **Configuration choices:** Notes prompt editing and `.com` normalization.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### 22. Sticky Note5

- **Type and role:** `n8n-nodes-base.stickyNote`; annotation for checking, routing, and loop control.
- **Configuration choices:** Summarizes API Ninjas checking, split, state update, and loop/exit.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual workflow start |  | Read sheet available | ## How it works<br>Manual Trigger starts a run. The workflow reads your Google Sheet (tabs "available" and "closed") to get domains already checked. Build initial state builds the alreadyChecked list and totalFound count.<br><br>Prepare prompt builds the prompt with your app context and passes alreadyChecked so the AI does not suggest the same names again. Generate names (OpenAI) returns 20 new names; Normalize to domains turns them into .com domains.<br><br>Check domain (API Ninjas) checks each name. Merge name and API result keeps only items with a valid API response. Filter available only and Closed only split results into available vs taken. Available domains go to Append to sheet available, taken ones to Append to sheet closed. State for next run updates the counters and alreadyChecked list. Exit or loop stops when you have at least 15 available domains or after 10 iterations (20 names per iteration), otherwise it loops back to Prepare prompt. |
| Read sheet available | Google Sheets | Read existing available domains from sheet | Manual Trigger | Trigger closed read | ## Start & load state<br><br>Reads sheets "available" and "closed", builds alreadyChecked and totalFound for the loop. |
| Trigger closed read | Code | Emit a dummy item to continue sequential state loading | Read sheet available | Read sheet closed | ## Start & load state<br><br>Reads sheets "available" and "closed", builds alreadyChecked and totalFound for the loop. |
| Read sheet closed | Google Sheets | Read existing unavailable domains from sheet | Trigger closed read | Build initial state | ## Start & load state<br><br>Reads sheets "available" and "closed", builds alreadyChecked and totalFound for the loop. |
| Build initial state | Code | Build initial state object from both sheets | Read sheet closed | Prepare prompt | ## Start & load state<br><br>Reads sheets "available" and "closed", builds alreadyChecked and totalFound for the loop. |
| Prepare prompt | Code | Build OpenAI prompts with exclusions and loop state | Build initial state; Exit or loop | Generate names (OpenAI) | ## Prompt & names<br><br>Edit Prepare prompt for your product; OpenAI returns 20 names; Normalize to domains makes them .com. |
| Generate names (OpenAI) | HTTP Request | Call OpenAI chat completions API | Prepare prompt | Normalize to domains | ## Prompt & names<br><br>Edit Prepare prompt for your product; OpenAI returns 20 names; Normalize to domains makes them .com. |
| Normalize to domains | Code | Parse AI JSON and convert names into `.com` domains | Generate names (OpenAI) | Check domain (API Ninjas) | ## Prompt & names<br><br>Edit Prepare prompt for your product; OpenAI returns 20 names; Normalize to domains makes them .com. |
| Check domain (API Ninjas) | HTTP Request | Check whether each `.com` domain is available | Normalize to domains | Merge name and API result | ## Check & route<br><br>API Ninjas checks each domain; results split into available/closed; state updated; loop or exit. |
| Merge name and API result | Code | Combine normalized names with API response and attach state | Check domain (API Ninjas) | Filter available only; State for next run; Closed only | ## Check & route<br><br>API Ninjas checks each domain; results split into available/closed; state updated; loop or exit. |
| Filter available only | Filter | Keep only available domains | Merge name and API result | Append to sheet available | ## Check & route<br><br>API Ninjas checks each domain; results split into available/closed; state updated; loop or exit. |
| State for next run | Code | Update cumulative loop state | Merge name and API result | Exit or loop | ## Check & route<br><br>API Ninjas checks each domain; results split into available/closed; state updated; loop or exit. |
| Closed only | Code | Keep only unavailable domains for storage | Merge name and API result | Append to sheet closed | ## Check & route<br><br>API Ninjas checks each domain; results split into available/closed; state updated; loop or exit. |
| Exit or loop | If | Stop or restart another generation round | State for next run | Prepare prompt (false branch) | ## Check & route<br><br>API Ninjas checks each domain; results split into available/closed; state updated; loop or exit. |
| Append to sheet available | Google Sheets | Append available domains to `available` tab | Filter available only |  | ## Write to sheets<br><br>Appends available domains to sheet "available", taken ones to sheet "closed". |
| Append to sheet closed | Google Sheets | Append unavailable domains to `closed` tab | Closed only |  | ## Write to sheets<br><br>Appends available domains to sheet "available", taken ones to sheet "closed". |
| Sticky Note | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a `Manual Trigger` node**.
   - Leave default configuration.
   - This is the workflow entry point.

3. **Create a Google spreadsheet** outside n8n.
   - Add two tabs:
     - `available`
     - `closed`
   - In both tabs, add header row:
     - `domain`
     - `name`

4. **Create Google Sheets credentials** in n8n.
   - Use Google Sheets OAuth2.
   - Ensure the connected Google account has edit access to the spreadsheet.

5. **Add a `Google Sheets` node named `Read sheet available`**.
   - Operation: read rows/default read behavior
   - Document: your spreadsheet ID
   - Sheet name: `available`
   - Enable **Continue On Fail**
   - Connect `Manual Trigger -> Read sheet available`

6. **Add a `Code` node named `Trigger closed read`**.
   - Paste:
     ```javascript
     return [{ json: {} }];
     ```
   - Connect `Read sheet available -> Trigger closed read`

7. **Add another `Google Sheets` node named `Read sheet closed`**.
   - Same spreadsheet ID
   - Sheet name: `closed`
   - Enable **Continue On Fail**
   - Connect `Trigger closed read -> Read sheet closed`

8. **Add a `Code` node named `Build initial state`**.
   - Paste:
     ```javascript
     let availableRows = [];
     let closedRows = [];
     try { availableRows = $('Read sheet available').all().map(i => i.json || i) || []; } catch (e) { }
     try { closedRows = $('Read sheet closed').all().map(i => i.json || i) || []; } catch (e) { }
     const totalFound = Array.isArray(availableRows) ? availableRows.length : 0;
     const dom = (r) => (r && (r.domain || r.name || r.Domain || r.Name));
     const fromAv = (Array.isArray(availableRows) ? availableRows : []).map(dom).filter(Boolean);
     const fromCl = (Array.isArray(closedRows) ? closedRows : []).map(dom).filter(Boolean);
     const alreadyChecked = [...new Set([...fromAv, ...fromCl])];
     return { json: { totalFound, alreadyChecked, iteration: 0 } };
     ```
   - Connect `Read sheet closed -> Build initial state`

9. **Add a `Code` node named `Prepare prompt`**.
   - Paste:
     ```javascript
     const input = $input.first().json;
     const totalFound = input.totalFound ?? 0;
     const alreadyChecked = Array.isArray(input.alreadyChecked) ? input.alreadyChecked : [];
     const iteration = input.iteration ?? 0;
     const excludeText = alreadyChecked.length > 0
       ? ` Do NOT suggest these names or domains (already checked, taken or already used): ${alreadyChecked.join(', ')}.`
       : '';
     const systemPrompt = `You are a naming expert. Generate short, distinctive, memorable brand names in English for a product or app. Use only letters, no numbers or hyphens. Each name must be under 14 characters. Return a valid JSON object with a single key "names" containing an array of exactly 20 strings.${excludeText}`;
     const userPrompt = `Generate 20 new, unique names for a product or app. Letters only, no numbers or hyphens, under 14 characters per name.${excludeText} Return ONLY valid JSON: { "names": ["Name1", "Name2", ...] }`;
     return { json: { systemPrompt, userPrompt, totalFound, alreadyChecked, iteration } };
     ```
   - Connect `Build initial state -> Prepare prompt`
   - Later, a loop will reconnect to this node.

10. **Optional but recommended:** customize the prompt text in `Prepare prompt`.
    - Add product category, tone, style, or market.
    - Keep the JSON output requirement intact.

11. **Create OpenAI credentials** for HTTP header auth.
    - Header name typically: `Authorization`
    - Header value: `Bearer YOUR_OPENAI_API_KEY`

12. **Add an `HTTP Request` node named `Generate names (OpenAI)`**.
    - Method: `POST`
    - URL: `https://api.openai.com/v1/chat/completions`
    - Authentication: Generic Credential Type
    - Generic Auth Type: HTTP Header Auth
    - Add header:
      - `Content-Type` = `application/json`
    - Send Body: enabled
    - Body type/specify body: JSON
    - JSON body:
      ```javascript
      ={{ JSON.stringify({ model: 'gpt-4o', temperature: 0.7, max_tokens: 2000, messages: [{ role: 'system', content: $json.systemPrompt }, { role: 'user', content: $json.userPrompt }], response_format: { type: 'json_object' } }) }}
      ```
    - Connect `Prepare prompt -> Generate names (OpenAI)`

13. **Add a `Code` node named `Normalize to domains`**.
    - Paste:
      ```javascript
      const openaiResponse = $json;
      if (!openaiResponse.choices || !openaiResponse.choices[0] || !openaiResponse.choices[0].message || !openaiResponse.choices[0].message.content) {
        throw new Error('OpenAI returned empty or invalid response');
      }
      const parsed = JSON.parse(openaiResponse.choices[0].message.content);
      const names = Array.isArray(parsed.names) ? parsed.names : [];
      if (names.length === 0) throw new Error('AI returned no names');
      const items = names.map(name => {
        const clean = String(name).trim().replace(/\s+/g, '').replace(/[^a-zA-Z]/g, '');
        if (!clean) return null;
        const domain = clean.toLowerCase() + '.com';
        return { json: { name: clean, domain } };
      }).filter(Boolean);
      return items;
      ```
    - Connect `Generate names (OpenAI) -> Normalize to domains`

14. **Create API Ninjas credentials** for HTTP header auth.
    - Header name typically: `X-Api-Key`
    - Header value: your API Ninjas key

15. **Add an `HTTP Request` node named `Check domain (API Ninjas)`**.
    - Method: `GET` or default request mode used by node
    - URL: `https://api.api-ninjas.com/v1/domain`
    - Authentication: Generic Credential Type
    - Generic Auth Type: HTTP Header Auth
    - Send Query Parameters: enabled
    - Query parameter:
      - `domain` = `={{ $json.domain }}`
    - In response options, enable behavior equivalent to:
      - `Never Error`
    - Enable **Continue On Fail**
    - Connect `Normalize to domains -> Check domain (API Ninjas)`

16. **Add a `Code` node named `Merge name and API result`**.
    - Paste:
      ```javascript
      const apiItems = $input.all();
      const normNode = $('Normalize to domains');
      const normItems = normNode.all();
      const state = $('Prepare prompt').last().json;
      const roundState = { totalFound: state.totalFound ?? 0, alreadyChecked: Array.isArray(state.alreadyChecked) ? state.alreadyChecked : [], iteration: state.iteration ?? 0 };
      const result = [];
      for (let i = 0; i < apiItems.length; i++) {
        const item = apiItems[i];
        const apiJson = item.json || {};
        if (typeof apiJson.available === 'boolean') {
          result.push({ json: { ...normItems[i].json, ...apiJson, _roundState: roundState } });
        }
      }
      return result;
      ```
    - Connect `Check domain (API Ninjas) -> Merge name and API result`

17. **Add a `Filter` node named `Filter available only`**.
    - Condition:
      - `{{ $json.available }}` equals `true`
    - Connect `Merge name and API result -> Filter available only`

18. **Add a `Google Sheets` node named `Append to sheet available`**.
    - Operation: `append`
    - Spreadsheet ID: same document
    - Sheet name: `available`
    - Use auto-mapping
    - Ensure columns include `domain` and `name`
    - Connect `Filter available only -> Append to sheet available`

19. **Add a `Code` node named `Closed only`**.
    - Paste:
      ```javascript
      const items = $input.all();
      return items.filter(i => i.json && i.json.available === false).map(i => ({ json: { domain: i.json.domain, name: i.json.name } }));
      ```
    - Connect `Merge name and API result -> Closed only`

20. **Add a `Google Sheets` node named `Append to sheet closed`**.
    - Operation: `append`
    - Spreadsheet ID: same document
    - Sheet name: `closed`
    - Use auto-mapping
    - Ensure columns include `domain` and `name`
    - Connect `Closed only -> Append to sheet closed`

21. **Add a `Code` node named `State for next run`**.
    - Paste:
      ```javascript
      const mergeItems = $input.all();
      const first = mergeItems[0] && mergeItems[0].json;
      const prev = first && first._roundState ? first._roundState : { totalFound: 0, alreadyChecked: [], iteration: 0 };
      const totalFound = prev.totalFound ?? 0;
      const alreadyChecked = prev.alreadyChecked ?? [];
      const iteration = prev.iteration ?? 0;
      const availableCount = mergeItems.filter(i => i.json && i.json.available === true).length;
      const domainsThisRound = mergeItems.map(i => i.json && i.json.domain).filter(Boolean);
      const newAlreadyChecked = [...new Set([...alreadyChecked, ...domainsThisRound])];
      const newTotalFound = totalFound + availableCount;
      const newIteration = iteration + 1;
      return { json: { totalFound: newTotalFound, alreadyChecked: newAlreadyChecked, iteration: newIteration } };
      ```
    - Connect `Merge name and API result -> State for next run`

22. **Add an `If` node named `Exit or loop`**.
    - Use OR logic between two conditions.
    - Condition 1:
      - `={{ $json.totalFound >= 15 }}` equals `true`
    - Condition 2:
      - `={{ $json.iteration > 10 }}` equals `true`
    - Connect `State for next run -> Exit or loop`

23. **Create the loop-back connection**.
    - Connect the **false** output of `Exit or loop` to `Prepare prompt`.
    - Leave the **true** output unconnected so the workflow ends when conditions are met.

24. **Test the workflow manually**.
    - Run from `Manual Trigger`.
    - Confirm:
      - OpenAI returns JSON with `names`
      - normalized names become `.com`
      - API Ninjas returns `available`
      - rows are appended to correct tabs
      - loop stops after 15 available results or iteration limit

25. **Optional visual documentation:** add sticky notes.
    - One for startup/load state
    - One for prompt/name generation
    - One for checking/routing
    - One for writing to sheets
    - One for credentials/setup

## Required Credentials Summary

- **Google Sheets OAuth2**
  - Used by:
    - `Read sheet available`
    - `Read sheet closed`
    - `Append to sheet available`
    - `Append to sheet closed`

- **OpenAI HTTP Header Auth**
  - Used by:
    - `Generate names (OpenAI)`
  - Typical header:
    - `Authorization: Bearer <OPENAI_API_KEY>`

- **API Ninjas HTTP Header Auth**
  - Used by:
    - `Check domain (API Ninjas)`
  - Typical header:
    - `X-Api-Key: <API_NINJAS_KEY>`

## Input/Output Expectations

- **Input at start:** none; manual execution only
- **Intermediate state object:**
  - `totalFound` number
  - `alreadyChecked` array of strings
  - `iteration` number
- **Generated name item:**
  - `name`
  - `domain`
- **Merged checked item:**
  - `name`
  - `domain`
  - `available`
  - optional API metadata
  - `_roundState`

## No Sub-workflows

This workflow does **not** invoke any sub-workflows and is not itself documented as a callable child workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Credentials required: OpenAI API key, API Ninjas API key, Google Sheets OAuth2 | Setup prerequisite |
| Spreadsheet must contain two tabs named `available` and `closed` with headers `domain, name` | Data persistence design |
| Prompt can be customized to reflect the target product, app category, style, or reference brands | `Prepare prompt` node |
| The workflow uses the Google Sheet as memory to avoid checking the same domains again across runs | Overall design |
| The loop ends when at least 15 available domains are found or after more than 10 iterations | Loop control logic |
| API Ninjas key is obtained from API Ninjas | https://api-ninjas.com/ |
| OpenAI chat completions endpoint used by the workflow | https://api.openai.com/v1/chat/completions |