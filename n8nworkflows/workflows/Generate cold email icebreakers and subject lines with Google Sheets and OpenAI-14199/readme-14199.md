Generate cold email icebreakers and subject lines with Google Sheets and OpenAI

https://n8nworkflows.xyz/workflows/generate-cold-email-icebreakers-and-subject-lines-with-google-sheets-and-openai-14199


# Generate cold email icebreakers and subject lines with Google Sheets and OpenAI

# 1. Workflow Overview

This workflow reads enriched lead records from a Google Sheet, identifies rows that still need personalization, generates a custom cold-email icebreaker and subject line with OpenAI, then writes the result back to the same sheet.

Primary use case:
- Personalizing outbound prospecting lists at scale
- Safely re-running the workflow without overwriting already processed rows
- Processing a controlled number of leads per execution

Logical blocks:

## 1.1 Input Reception and Lead Loading
The workflow starts from a Manual Trigger, reads all rows from a Google Sheet, and loads the lead dataset into n8n.

## 1.2 Lead Filtering and Throughput Control
It filters rows where the `icebreakers` field is empty or the `subjectLine` field is empty, then caps processing to 200 leads per run.

## 1.3 Sequential Processing Loop
It sends the filtered items into a `Split In Batches` loop so each lead is handled one at a time.

## 1.4 Conditional Check and AI Generation
For each row, it verifies again that the lead still needs personalization, then sends prospect data to OpenAI using a few-shot prompt that requests structured JSON output.

## 1.5 Result Persistence and Rate Limiting
The generated result is written back to the original Google Sheet row using `row_number` as the matching key. The workflow then waits 1 second before requesting the next batch item.

## 1.6 Documentation and Operator Guidance
Several Sticky Notes explain setup, required columns, customization options, and operational warnings.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Lead Loading

### Overview
This block starts the workflow manually and retrieves all lead rows from the configured Google Sheet. It forms the data source for the rest of the workflow.

### Nodes Involved
- `When clicking 'Execute workflow'`
- `Read Lead Sheet`

### Node Details

#### When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for ad hoc execution from the n8n editor.
- **Configuration choices:** No parameters are required. The node runs only when a user clicks Execute.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Read Lead Sheet`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No runtime failure expected in normal use.
  - Not suitable for unattended automation; must be replaced with another trigger for scheduled or event-driven runs.
- **Sub-workflow reference:** None.

#### Read Lead Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a spreadsheet.
- **Configuration choices:**
  - Uses Google Sheets OAuth2 credentials.
  - Reads from spreadsheet ID: `your-google-sheet-id` placeholder, which must be replaced.
  - Targets sheet `gid=0` / `Sheet1`.
  - No explicit operation is set in the JSON because this node is using the default read behavior for the Google Sheets node version.
- **Key expressions or variables used:** None in configuration.
- **Input and output connections:**
  - Input: `When clicking 'Execute workflow'`
  - Output: `Filter Empty Rows`
- **Version-specific requirements:** Type version `4.7`; requires a valid Google Sheets OAuth2 credential compatible with Google Sheets v4 node behavior.
- **Edge cases or potential failure types:**
  - OAuth2 authentication failure
  - Spreadsheet ID invalid or inaccessible
  - Wrong tab selected
  - Missing required columns later in the workflow
  - Large sheet sizes may increase runtime
- **Sub-workflow reference:** None.

---

## 2.2 Lead Filtering and Throughput Control

### Overview
This block removes leads that are already fully processed and limits the number of rows handled in a single execution. It makes the workflow safe to re-run and helps control API cost and rate.

### Nodes Involved
- `Filter Empty Rows`
- `Limit to 200 Leads`

### Node Details

#### Filter Empty Rows
- **Type and technical role:** `n8n-nodes-base.filter`; filters incoming items based on field conditions.
- **Configuration choices:**
  - Uses `or` combinator.
  - Passes rows where:
    - `icebreakers` is empty, or
    - `subjectLine` equals an empty string
  - This means any row missing either output field is eligible for generation.
- **Key expressions or variables used:**
  - `{{ $json.icebreakers }}`
  - `{{ $json.subjectLine }}`
- **Input and output connections:**
  - Input: `Read Lead Sheet`
  - Output: `Limit to 200 Leads`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the sheet column names differ from `icebreakers` or `subjectLine`, filtering will behave incorrectly.
  - Null values vs empty strings can behave differently depending on how Sheets returns data.
  - Rows with whitespace-only content are not explicitly trimmed here.
- **Sub-workflow reference:** None.

#### Limit to 200 Leads
- **Type and technical role:** `n8n-nodes-base.limit`; caps the number of items passed forward.
- **Configuration choices:**
  - `maxItems = 200`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Filter Empty Rows`
  - Output: `Process One by One`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No real failure mode unless upstream data is malformed.
  - If more than 200 rows need work, remaining rows wait for the next execution.
- **Sub-workflow reference:** None.

---

## 2.3 Sequential Processing Loop

### Overview
This block ensures leads are processed one at a time rather than in parallel. That helps reduce pressure on APIs and supports the loop-back mechanism after each write.

### Nodes Involved
- `Process One by One`

### Node Details

#### Process One by One
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through items in batches.
- **Configuration choices:**
  - Used as a one-by-one loop controller.
  - `reset: false`
  - Since no explicit batch size is shown, the intended pattern is the default iterative batching behavior with loopback.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input:
    - `Limit to 200 Leads`
    - loopback from `Rate Limit Delay`
  - Output:
    - Main output index 1 goes to `Needs Icebreaker?`
    - Output index 0 is unused in this workflow
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Misunderstanding output branches can break the loop if rebuilt incorrectly.
  - If loopback is removed, only the first batch may process.
  - If upstream returns zero items, nothing further happens.
- **Sub-workflow reference:** None.

---

## 2.4 Conditional Check and AI Generation

### Overview
This block re-checks whether the current row still needs personalization and then prompts OpenAI to generate structured JSON containing the icebreaker and subject line. The second check protects against stale or unexpectedly changed data during processing.

### Nodes Involved
- `Needs Icebreaker?`
- `Generate Icebreaker`

### Node Details

#### Needs Icebreaker?
- **Type and technical role:** `n8n-nodes-base.if`; conditional gate.
- **Configuration choices:**
  - Uses `or` combinator.
  - True when:
    - `icebreakers` is empty, or
    - `subjectLine` equals empty string
- **Key expressions or variables used:**
  - `{{ $json.icebreakers }}`
  - `{{ $json.subjectLine }}`
- **Input and output connections:**
  - Input: `Process One by One`
  - Output:
    - True branch: `Generate Icebreaker`
    - False branch: not connected
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Same schema sensitivity as the earlier filter node
  - False branch drops items silently because nothing is connected
- **Sub-workflow reference:** None.

#### Generate Icebreaker
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; calls OpenAI chat generation through the LangChain-based OpenAI node.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - `jsonOutput: true`
  - Uses a multi-message prompt:
    - system message defines the assistant as a helpful writing assistant
    - instruction message defines required JSON schema:
      - `verdict`
      - `subjectLine`
      - `icebreaker`
      - `shortenedCompanyName`
    - includes few-shot examples
    - final user message maps lead fields from the sheet into the prompt
  - The prompt explicitly requests a personalized one-line email icebreaker and subject line.
- **Key expressions or variables used:**
  - `{{ $json.first_name }}`
  - `{{ $json.last_name }}`
  - `{{ $json.job_title }}`
  - `{{ $json.company }}`
  - `{{ $json.linkedin_industry }}`
  - `{{ $json.location }}`
  - `{{ $json.linkedin_company_employee_count }}`
  - `{{ $json.linkedin_founded_year }}`
  - `{{ $json.summary }}`
  - `{{ $json.linkedin_description }}`
  - `{{ $json.linkedin_specialities }}`
- **Input and output connections:**
  - Input: `Needs Icebreaker?`
  - Output: `Write Results to Sheet`
- **Version-specific requirements:** Type version `1.8`; requires compatible OpenAI credentials and availability of `gpt-4.1-mini` in the connected OpenAI account.
- **Edge cases or potential failure types:**
  - OpenAI authentication or quota errors
  - Model unavailability
  - Rate limiting
  - Response may not perfectly match expected JSON schema despite `jsonOutput: true`
  - Missing sheet fields can reduce personalization quality or result in odd prompt text
  - The prompt asks for `subjectLine`, but the write node currently only stores `icebreaker`
  - If the model returns nested content differently, downstream expressions may fail
- **Sub-workflow reference:** None.

---

## 2.5 Result Persistence and Rate Limiting

### Overview
This block saves generated content back into the source spreadsheet and inserts a small delay before the next loop iteration. It supports idempotent updates by matching on `row_number`.

### Nodes Involved
- `Write Results to Sheet`
- `Rate Limit Delay`

### Node Details

#### Write Results to Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates rows in the spreadsheet.
- **Configuration choices:**
  - Operation: `update`
  - Spreadsheet ID: `your-google-sheet-id` placeholder
  - Sheet: `gid=0` / `Sheet1`
  - Matching column: `row_number`
  - Mapped columns:
    - `row_number = {{ $('Needs Icebreaker?').item.json.row_number }}`
    - `icebreakers = {{ $json.message.content.icebreaker }}`
  - Type conversion is disabled.
- **Key expressions or variables used:**
  - `{{ $('Needs Icebreaker?').item.json.row_number }}`
  - `{{ $json.message.content.icebreaker }}`
- **Input and output connections:**
  - Input: `Generate Icebreaker`
  - Output: `Rate Limit Delay`
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or potential failure types:**
  - Spreadsheet update permission failure
  - No matching `row_number` found
  - `row_number` missing from source data
  - Expression path may fail if the OpenAI node returns a different object shape
  - Only `icebreakers` is written; `subjectLine` is not mapped despite being generated and mentioned in sticky notes
- **Sub-workflow reference:** None.

#### Rate Limit Delay
- **Type and technical role:** `n8n-nodes-base.wait`; delays execution before looping.
- **Configuration choices:**
  - Wait amount: `1`
  - In context, this is used as a 1-second delay between processed leads.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Write Results to Sheet`
  - Output: `Process One by One`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If the Wait node is reconfigured incorrectly, loop pacing may be too aggressive or too slow.
  - Long workflows can accumulate total runtime due to per-item waits.
- **Sub-workflow reference:** None.

---

## 2.6 Documentation and Operator Guidance

### Overview
These nodes are non-executable annotations placed on the canvas. They explain purpose, setup steps, customization options, and warnings.

### Nodes Involved
- `Main Description`
- `Section - Load`
- `Section - Generate`
- `Section - Save`
- `Warning`

### Node Details

#### Main Description
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node.
- **Configuration choices:** Large explanatory note with workflow overview, setup checklist, required columns, and customization suggestions.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; informational only.
- **Sub-workflow reference:** None.

#### Section - Load
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for loading/filtering area.
- **Configuration choices:** Describes the lead-loading stage.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Section - Generate
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for generation area.
- **Configuration choices:** Describes the one-by-one loop and OpenAI generation stage.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Section - Save
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for persistence and loopback area.
- **Configuration choices:** Describes saving results and throttling.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Warning
- **Type and technical role:** `n8n-nodes-base.stickyNote`; operational warning note.
- **Configuration choices:** Warns users to connect credentials and set spreadsheet IDs.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow entry point |  | Read Lead Sheet | ### Load leads<br>Reads the enriched lead sheet, filters to rows missing icebreakers or subject lines, and caps at 200 per run. |
| Read Lead Sheet | Google Sheets | Load all lead rows from the source sheet | When clicking 'Execute workflow' | Filter Empty Rows | ### Load leads<br>Reads the enriched lead sheet, filters to rows missing icebreakers or subject lines, and caps at 200 per run.<br>**Dont forget to** Connect your Google Sheets and OpenAI credentials, and set the correct spreadsheet ID in both sheet nodes before running. |
| Filter Empty Rows | Filter | Keep only rows missing icebreaker or subject line | Read Lead Sheet | Limit to 200 Leads | ### Load leads<br>Reads the enriched lead sheet, filters to rows missing icebreakers or subject lines, and caps at 200 per run. |
| Limit to 200 Leads | Limit | Restrict processing volume per execution | Filter Empty Rows | Process One by One | ### Load leads<br>Reads the enriched lead sheet, filters to rows missing icebreakers or subject lines, and caps at 200 per run. |
| Process One by One | Split In Batches | Iterate through leads sequentially | Limit to 200 Leads; Rate Limit Delay | Needs Icebreaker? | ### Generate icebreakers<br>Loops through each lead one by one. Checks the row still needs an icebreaker, then calls OpenAI with few-shot examples to produce personalized copy. |
| Needs Icebreaker? | If | Re-check whether the current row still needs generation | Process One by One | Generate Icebreaker | ### Generate icebreakers<br>Loops through each lead one by one. Checks the row still needs an icebreaker, then calls OpenAI with few-shot examples to produce personalized copy. |
| Generate Icebreaker | OpenAI (LangChain) | Generate personalized JSON output for the lead | Needs Icebreaker? | Write Results to Sheet | ### Generate icebreakers<br>Loops through each lead one by one. Checks the row still needs an icebreaker, then calls OpenAI with few-shot examples to produce personalized copy. |
| Write Results to Sheet | Google Sheets | Update the original row with generated content | Generate Icebreaker | Rate Limit Delay | ### Save & loop<br>Writes the generated icebreaker and subject line back to the same row via row_number match. Waits 1s between calls to avoid API throttling, then loops to the next lead.<br>**Dont forget to** Connect your Google Sheets and OpenAI credentials, and set the correct spreadsheet ID in both sheet nodes before running. |
| Rate Limit Delay | Wait | Pause between items before requesting next batch | Write Results to Sheet | Process One by One | ### Save & loop<br>Writes the generated icebreaker and subject line back to the same row via row_number match. Waits 1s between calls to avoid API throttling, then loops to the next lead. |
| Main Description | Sticky Note | General workflow documentation |  |  | ## Email List Personalization - Icebreaker & Subject Line Generator<br><br>Reads an enriched lead list from Google Sheets, generates a personalized icebreaker and subject line for each lead using OpenAI, and writes results back to the same sheet.<br><br>### How it works<br><br>1. Reads all rows from your enriched lead sheet<br>2. Filters to only rows missing an icebreaker or subject line (safe to re-run)<br>3. Caps at 200 leads per run, then loops through one by one<br>4. OpenAI generates a JSON response with icebreaker, subject line, shortened company name, and a verdict flag<br>5. Writes results back to the same row via row_number match<br><br>### Setup<br><br>- [ ] Connect Google Sheets OAuth2 in both sheet nodes<br>- [ ] Set your spreadsheet ID in "Read Lead Sheet" and "Write Results to Sheet"<br>- [ ] Connect your OpenAI API key<br>- [ ] Required columns: first_name, last_name, email, job_title, company, linkedin_industry, location, summary, linkedin_description, linkedin_specialities, linkedin_company_employee_count, linkedin_founded_year, icebreakers, subjectLine, row_number<br><br>### Customization<br><br>- Replace Manual Trigger with Schedule or Google Drive Trigger for automation<br>- Adjust the Limit node to match your daily send volume<br>- Swap GPT-4.1-mini (~$0.002/lead) for GPT-4o (~$0.01/lead) on premium campaigns<br>- Edit the few-shot examples in the OpenAI node to match your voice |
| Section - Load | Sticky Note | Visual section label for load stage |  |  | ### Load leads<br>Reads the enriched lead sheet, filters to rows missing icebreakers or subject lines, and caps at 200 per run. |
| Section - Generate | Sticky Note | Visual section label for generation stage |  |  | ### Generate icebreakers<br>Loops through each lead one by one. Checks the row still needs an icebreaker, then calls OpenAI with few-shot examples to produce personalized copy. |
| Section - Save | Sticky Note | Visual section label for save/loop stage |  |  | ### Save & loop<br>Writes the generated icebreaker and subject line back to the same row via row_number match. Waits 1s between calls to avoid API throttling, then loops to the next lead. |
| Warning | Sticky Note | Visual operational warning |  |  | **Dont forget to** Connect your Google Sheets and OpenAI credentials, and set the correct spreadsheet ID in both sheet nodes before running. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Email List Personalization - Icebreaker And Subject Line Generator`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name: `When clicking 'Execute workflow'`

3. **Add a Google Sheets read node**
   - Node type: `Google Sheets`
   - Name: `Read Lead Sheet`
   - Connect it from the Manual Trigger.
   - Configure Google Sheets OAuth2 credentials.
   - Select your spreadsheet using its ID.
   - Select the worksheet/tab, here represented by `gid=0` / `Sheet1`.
   - Use the read mode that returns all rows.

4. **Prepare the source sheet structure**
   - Ensure the sheet contains at least these columns:
     - `first_name`
     - `last_name`
     - `email`
     - `job_title`
     - `company`
     - `linkedin_industry`
     - `location`
     - `summary`
     - `linkedin_description`
     - `linkedin_specialities`
     - `linkedin_company_employee_count`
     - `linkedin_founded_year`
     - `icebreakers`
     - `subjectLine`
     - `row_number`
   - `row_number` must uniquely identify each row for updates.

5. **Add a Filter node**
   - Node type: `Filter`
   - Name: `Filter Empty Rows`
   - Connect it from `Read Lead Sheet`.
   - Set condition logic to `OR`.
   - Add condition 1:
     - Left value: `{{ $json.icebreakers }}`
     - Operator: `empty`
   - Add condition 2:
     - Left value: `{{ $json.subjectLine }}`
     - Operator: `equals`
     - Right value: empty string
   - Purpose: keep rows missing either field.

6. **Add a Limit node**
   - Node type: `Limit`
   - Name: `Limit to 200 Leads`
   - Connect it from `Filter Empty Rows`.
   - Set `Max Items` to `200`.

7. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name: `Process One by One`
   - Connect it from `Limit to 200 Leads`.
   - Keep reset disabled.
   - Use it as the loop controller for per-item processing.

8. **Add an If node**
   - Node type: `If`
   - Name: `Needs Icebreaker?`
   - Connect it from the batch output used for iterating items.
   - Set condition logic to `OR`.
   - Add condition 1:
     - Left value: `{{ $json.icebreakers }}`
     - Operator: `is empty`
   - Add condition 2:
     - Left value: `{{ $json.subjectLine }}`
     - Operator: `equals`
     - Right value: empty string
   - Leave the false branch unconnected if you want to skip already complete rows silently.

9. **Add an OpenAI node**
   - Node type: `OpenAI` from the LangChain/OpenAI integration
   - Name: `Generate Icebreaker`
   - Connect it from the true output of `Needs Icebreaker?`
   - Configure OpenAI credentials.
   - Set model to `gpt-4.1-mini`
   - Enable structured/JSON output if available as `jsonOutput`.

10. **Configure the OpenAI messages**
    - Add a system message:
      - `You are a helpful, intelligent writing assistant.`
    - Add an instruction message telling the model to:
      - Generate a one-line personalized cold email icebreaker
      - Also generate a subject line
      - Return JSON with:
        - `verdict`
        - `subjectLine`
        - `icebreaker`
        - `shortenedCompanyName`
      - Use a concise, laconic tone
      - Return `"false"` for `verdict` if the lead looks like a company rather than a person
      - Shorten company names and locations where reasonable
    - Add the two few-shot examples shown in the original workflow to guide tone and formatting.
    - Add the final dynamic lead prompt using expressions:
      - `Name: {{ $json.first_name }} {{ $json.last_name }}`
      - `Title: {{ $json.job_title }}`
      - `Company: {{ $json.company }}`
      - `Industry: {{ $json.linkedin_industry }}`
      - `Location: {{ $json.location }}`
      - `Employees: {{ $json.linkedin_company_employee_count }}`
      - `Founded: {{ $json.linkedin_founded_year }}`
      - `About them: {{ $json.summary }}`
      - `About the company: {{ $json.linkedin_description }}`
      - `Specialties: {{ $json.linkedin_specialities }}`

11. **Add a Google Sheets update node**
    - Node type: `Google Sheets`
    - Name: `Write Results to Sheet`
    - Connect it from `Generate Icebreaker`
    - Use the same Google Sheets OAuth2 credential as the read node.
    - Set operation to `Update`
    - Select the same spreadsheet and same sheet/tab.
    - Set matching column to `row_number`

12. **Configure the update mapping**
    - Map `row_number` to:
      - `{{ $('Needs Icebreaker?').item.json.row_number }}`
    - Map `icebreakers` to:
      - `{{ $json.message.content.icebreaker }}`
    - Important: the provided workflow does **not** map `subjectLine`, even though it is generated.
    - If you want the workflow to match its stated intent, also add:
      - `subjectLine = {{ $json.message.content.subjectLine }}`
    - Leave type conversion disabled if you want exact values preserved.

13. **Add a Wait node**
    - Node type: `Wait`
    - Name: `Rate Limit Delay`
    - Connect it from `Write Results to Sheet`
    - Set delay amount to `1` second.

14. **Create the loopback**
    - Connect `Rate Limit Delay` back into `Process One by One`
    - This allows the next item to be processed after each delay.

15. **Verify the batch routing**
    - Ensure `Process One by One` sends the current item forward to `Needs Icebreaker?`
    - In this exported workflow, the item path is connected from the second output branch of `Split In Batches`, so reproduce that same routing pattern carefully.

16. **Add optional Sticky Notes for documentation**
    - Add notes describing:
      - workflow purpose
      - required credentials
      - required columns
      - load/generate/save sections
      - customization ideas

17. **Configure credentials**
    - **Google Sheets OAuth2**
      - Must have read/write access to the spreadsheet.
      - Apply to both Google Sheets nodes.
    - **OpenAI API**
      - Must allow use of the selected model.
      - Ensure billing/quota is enabled.

18. **Test with a small dataset**
    - Start with a few rows.
    - Verify:
      - rows are read correctly
      - only missing rows are selected
      - OpenAI returns valid JSON
      - updates match the correct row via `row_number`

19. **Check for schema consistency**
    - Confirm the OpenAI node output path matches the expression used in `Write Results to Sheet`.
    - If your OpenAI node version outputs content in a different structure, update expressions accordingly.

20. **Consider production improvements**
    - Replace Manual Trigger with:
      - `Schedule Trigger` for recurring runs, or
      - another event trigger
    - Add error handling for:
      - OpenAI malformed JSON
      - Google Sheets write failure
      - rows with missing names or company info
    - Optionally add a field to store:
      - `subjectLine`
      - `verdict`
      - `shortenedCompanyName`
      - processing timestamp

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Email List Personalization - Icebreaker & Subject Line Generator | Workflow title/purpose |
| Reads an enriched lead list from Google Sheets, generates a personalized icebreaker and subject line for each lead using OpenAI, and writes results back to the same sheet. | Main workflow description |
| Connect Google Sheets OAuth2 in both sheet nodes. | Setup note |
| Set your spreadsheet ID in `Read Lead Sheet` and `Write Results to Sheet`. | Setup note |
| Connect your OpenAI API key. | Setup note |
| Required columns: `first_name`, `last_name`, `email`, `job_title`, `company`, `linkedin_industry`, `location`, `summary`, `linkedin_description`, `linkedin_specialities`, `linkedin_company_employee_count`, `linkedin_founded_year`, `icebreakers`, `subjectLine`, `row_number` | Data model note |
| Replace Manual Trigger with Schedule or Google Drive Trigger for automation. | Customization note |
| Adjust the Limit node to match your daily send volume. | Operational note |
| Swap GPT-4.1-mini (~$0.002/lead) for GPT-4o (~$0.01/lead) on premium campaigns. | Cost/quality tuning note |
| Edit the few-shot examples in the OpenAI node to match your voice. | Prompt customization note |
| Important implementation discrepancy: the workflow description says it writes both icebreaker and subject line back to the sheet, but the actual `Write Results to Sheet` node only maps `icebreakers`. | Critical technical note |