Send end-of-life software alerts using NocoDB, endoflife.date, and Slack

https://n8nworkflows.xyz/workflows/send-end-of-life-software-alerts-using-nocodb--endoflife-date--and-slack-14309


# Send end-of-life software alerts using NocoDB, endoflife.date, and Slack

# 1. Workflow Overview

This workflow monitors software end-of-life status for projects stored in NocoDB, refreshes lifecycle data from the public `endoflife.date` API, stores the normalized results in NocoDB, and posts structured alerts to Slack when a project uses software that is already EOL, becomes EOL today, or will become EOL soon.

It has two explicit entry points:

- **Create Tables**: one-time manual bootstrap path that creates the required NocoDB tables and relation.
- **Run daily**: scheduled operational path that refreshes lifecycle data and sends notifications.

## 1.1 Initial Setup and Table Provisioning
This block creates the required NocoDB schema: `EOLSoftware`, `EOLDates`, `EOLProjects`, plus a relation linking projects to software date records.

## 1.2 Runtime Configuration
A small config block defines the lead time for notifications, in days.

## 1.3 Software Intake and Iteration
The workflow loads the list of tracked software from NocoDB and iterates through it one software item at a time.

## 1.4 Fetch and Normalize endoflife.date Data
For each software slug, the workflow queries `https://endoflife.date/api/<software>.json`, enriches the records, renames keys into human-readable field names, and prepares a merge key for synchronization.

## 1.5 Sync into NocoDB EOLDates
The workflow loads existing `EOLDates` rows, matches incoming records by a synthetic `key`, converts boolean false date values into `null`, and either updates existing rows or inserts new ones. A wait node throttles the loop to reduce API pressure.

## 1.6 Project Evaluation
After all software items are processed, the workflow loads projects, fetches the linked EOL date rows per project, consolidates them into a single array, and computes whether any record is:
- already past EOL,
- due today,
- or due within the configured threshold.

## 1.7 Filtering and Slack Notification
Projects with no relevant EOL issues are discarded. Projects with issues trigger a Slack block message grouped by severity.

---

# 2. Block-by-Block Analysis

## 2.1 Initial Setup and Table Provisioning

**Overview:**  
This one-time bootstrap block creates the NocoDB tables required by the workflow and then creates the relationship field linking projects to lifecycle-date rows. It is intended to be triggered manually before the recurring schedule is used.

**Nodes Involved:**  
- Create Tables
- NocoDB Config
- Create NocoDB EOL Software Table
- Create NocoDB EOL Dates Table
- Create NocoDB EOL Projects Table
- Create relation between tables

### Node Details

#### Create Tables
- **Type and role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for one-time setup.
- **Configuration choices:** No parameters; it simply starts the provisioning path.
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: none
  - Output: `NocoDB Config`
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** None by itself; failures happen downstream.

#### NocoDB Config
- **Type and role:** `n8n-nodes-base.set`  
  Stores NocoDB base metadata needed by the HTTP metadata API calls.
- **Configuration choices:** Creates two fields:
  - `Base ID`
  - `NocoDB Url`
  Both are empty by default and must be filled manually.
- **Key expressions or variables used:** Static assignment.
- **Input / output connections:**  
  - Input: `Create Tables`
  - Output: `Create NocoDB EOL Software Table`
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**  
  - Empty `Base ID` or `NocoDB Url` causes invalid API endpoints.
  - Missing trailing slash in `NocoDB Url` may break URL construction depending on input format.
- **Sub-workflow reference:** None.

#### Create NocoDB EOL Software Table
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Creates table `EOLSoftware` in NocoDB using the metadata API.
- **Configuration choices:**  
  - Method: `POST`
  - URL uses `NocoDB Url` and `Base ID`
  - JSON body defines:
    - table title/name `EOLSoftware`
    - one `SingleLineText` field: `Software`
  - Auth uses predefined NocoDB API token credential.
- **Key expressions or variables used:**  
  - `{{ $json["NocoDB Url"] }}`
  - `{{ $('NocoDB Config').first().json['Base ID'] }}`
- **Input / output connections:**  
  - Input: `NocoDB Config`
  - Output: `Create NocoDB EOL Dates Table`
- **Version-specific requirements:** HTTP Request v4.3.
- **Edge cases / failures:**  
  - NocoDB auth/token invalid.
  - Base ID not found.
  - Table already exists may return an API error.
  - URL formatting issues.
- **Sub-workflow reference:** None.

#### Create NocoDB EOL Dates Table
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Creates table `EOLDates`, which stores normalized lifecycle records from endoflife.date.
- **Configuration choices:**  
  - Method: `POST`
  - JSON body creates fields:
    - `key`
    - `Software`
    - `Version`
    - `Latest Subversion`
    - `Subversion Release`
    - `End Of Life`
    - `End Of Support`
  - All are `SingleLineText`.
- **Key expressions or variables used:**  
  - `{{ $('NocoDB Config').first().json['NocoDB Url'] }}`
  - `{{ $('NocoDB Config').first().json['Base ID'] }}`
- **Input / output connections:**  
  - Input: `Create NocoDB EOL Software Table`
  - Output: `Create NocoDB EOL Projects Table`
- **Version-specific requirements:** HTTP Request v4.3.
- **Edge cases / failures:**  
  - Duplicate table creation.
  - Wrong NocoDB metadata API path.
  - Data types are all text, so date handling later depends on date-string consistency.
- **Sub-workflow reference:** None.

#### Create NocoDB EOL Projects Table
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Creates the `EOLProjects` table.
- **Configuration choices:**  
  - Method: `POST`
  - JSON body defines one field:
    - `Project Name` as `SingleLineText`
- **Key expressions or variables used:**  
  - `{{ $('NocoDB Config').first().json['NocoDB Url'] }}`
  - `{{ $('NocoDB Config').first().json['Base ID'] }}`
- **Input / output connections:**  
  - Input: `Create NocoDB EOL Dates Table`
  - Output: `Create relation between tables`
- **Version-specific requirements:** HTTP Request v4.3.
- **Edge cases / failures:** Same as above.
- **Sub-workflow reference:** None.

#### Create relation between tables
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Adds a relation field named `EOLDates` to the projects table so projects can link to EOLDates records.
- **Configuration choices:**  
  - Method: `POST`
  - Endpoint targets `/fields` for the projects table created immediately before
  - JSON body defines:
    - field title `EOLDates`
    - type `Links`
    - relation type `hm`
    - related table id from `Create NocoDB EOL Dates Table`
- **Key expressions or variables used:**  
  - `{{ $('NocoDB Config').first().json['NocoDB Url'] }}`
  - `{{ $('NocoDB Config').first().json['Base ID'] }}`
  - `{{ $json.id }}` from previous node
  - `{{ $('Create NocoDB EOL Dates Table').first().json['id'] }}`
- **Input / output connections:**  
  - Input: `Create NocoDB EOL Projects Table`
  - Output: none
- **Version-specific requirements:** HTTP Request v4.3.
- **Edge cases / failures:**  
  - If the previous node output is not the projects table object, `$json.id` may be wrong.
  - Relation type semantics may differ between NocoDB versions.
  - Existing field title conflict.
- **Sub-workflow reference:** None.

---

## 2.2 Runtime Configuration

**Overview:**  
This block holds the notification threshold used later when evaluating incoming EOL events. It is the main user-adjustable runtime setting.

**Nodes Involved:**  
- Run daily
- Config

### Node Details

#### Run daily
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Scheduled entry point for normal operation.
- **Configuration choices:**  
  - Runs daily at hour `7`
  - Minute is default/empty in the exported config
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: none
  - Output: `Config`
- **Version-specific requirements:** Schedule Trigger v1.2.
- **Edge cases / failures:**  
  - Timezone depends on workflow/server settings.
  - If minute is unset, n8n resolves it using its own UI/runtime defaults.
- **Sub-workflow reference:** None.

#### Config
- **Type and role:** `n8n-nodes-base.set`  
  Stores `Days before EOL`.
- **Configuration choices:**  
  - Assigns `Days before EOL = 31`
- **Key expressions or variables used:** Static number assignment.
- **Input / output connections:**  
  - Input: `Run daily`
  - Output: `Get software to check`
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**  
  - Negative or zero values would produce illogical alerts.
  - Large values may generate too many alerts.
- **Sub-workflow reference:** None.

---

## 2.3 Software Intake and Iteration

**Overview:**  
This block loads tracked software names from NocoDB and iterates over them individually. The loop fans out into both lifecycle API fetching and local row lookup.

**Nodes Involved:**  
- Get software to check
- Each Software Once

### Node Details

#### Get software to check
- **Type and role:** `n8n-nodes-base.nocoDb`  
  Reads all rows from the `EOLSoftware` table.
- **Configuration choices:**  
  - Operation: `getAll`
  - `returnAll: true`
  - Uses project ID `pksfpoc943gwhvy`
  - Table ID `mdxtflijkbf3po6`
  - Auth: NocoDB API token
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: `Config`
  - Output: `Each Software Once`
- **Version-specific requirements:** NocoDB node v3.
- **Edge cases / failures:**  
  - Invalid credentials or project/table IDs.
  - Empty table causes no downstream lifecycle refresh.
  - Software names must match endoflife.date slugs; otherwise the API request fails later.
- **Sub-workflow reference:** None.

#### Each Software Once
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates over one software row at a time and controls looping.
- **Configuration choices:**  
  - Default split behavior; no extra options set.
- **Key expressions or variables used:** Downstream nodes reference `$("Each Software Once").item.json.Software`.
- **Input / output connections:**  
  - Input: `Get software to check`
  - Output 0:
    - `Get EOLProjects`
    - `Get Current Rows`
    - `Get EOL data from endoflife.date`
  - Input loop-back from `Wait`
- **Version-specific requirements:** Split In Batches v3.
- **Edge cases / failures:**  
  - If no items come in, downstream blocks never execute.
  - Loop structure means project evaluation starts only after the batch loop completes through the node’s completion branch behavior.
- **Sub-workflow reference:** None.

---

## 2.4 Fetch and Normalize endoflife.date Data

**Overview:**  
For each software slug, this block pulls lifecycle data from `endoflife.date`, derives a unique key, and renames fields into the structure expected by NocoDB.

**Nodes Involved:**  
- Get EOL data from endoflife.date
- Add Key and Software
- Rename Keys

### Node Details

#### Get EOL data from endoflife.date
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Fetches lifecycle JSON for one software product.
- **Configuration choices:**  
  - URL: `https://endoflife.date/api/<software>.json`
  - Software name is lowercased from current loop item.
  - No authentication.
- **Key expressions or variables used:**  
  - `{{ $json.Software.toLowerCase() }}`
- **Input / output connections:**  
  - Input: `Each Software Once`
  - Output: `Add Key and Software`
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**  
  - Software slug mismatch leads to 404 or unexpected content.
  - API downtime or rate limiting.
  - Unexpected schema changes in endoflife.date responses.
- **Sub-workflow reference:** None.

#### Add Key and Software
- **Type and role:** `n8n-nodes-base.set`  
  Adds workflow-specific metadata to each lifecycle record.
- **Configuration choices:**  
  - Keeps other fields
  - Adds:
    - `key = <Software><cycle>`
    - `Software = <software from loop item>`
- **Key expressions or variables used:**  
  - `{{ $("Each Software Once").item.json.Software }}{{ $json.cycle }}`
  - `{{ $("Each Software Once").item.json.Software }}`
- **Input / output connections:**  
  - Input: `Get EOL data from endoflife.date`
  - Output: `Rename Keys`
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**  
  - If `cycle` is missing, keys become malformed.
  - Concatenated keys may collide if software naming is inconsistent.
- **Sub-workflow reference:** None.

#### Rename Keys
- **Type and role:** `n8n-nodes-base.renameKeys`  
  Converts API field names into human-readable column names aligned with the NocoDB table.
- **Configuration choices:** Renames:
  - `cycle` → `Version`
  - `latest` → `Latest Subversion`
  - `latestReleaseDate` → `Subversion Release`
  - `eol` → `End Of Life`
  - `lts` → `Long Term Support`
  - `support` → `End Of Support`
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: `Add Key and Software`
  - Output: `Merge`
- **Version-specific requirements:** Rename Keys node v1.
- **Edge cases / failures:**  
  - Missing source keys produce absent target fields.
  - `Long Term Support` is created but not stored in the provisioned `EOLDates` schema, so it may be ignored during automapping depending on NocoDB behavior.
- **Sub-workflow reference:** None.

---

## 2.5 Sync into NocoDB EOLDates

**Overview:**  
This block combines fresh API data with current rows from `EOLDates`, normalizes invalid date values, and decides whether each record should be updated or created.

**Nodes Involved:**  
- Get Current Rows
- Make ID Key
- Merge
- Convert bool to null
- If ID exists
- Update Data
- Insert New
- Wait

### Node Details

#### Get Current Rows
- **Type and role:** `n8n-nodes-base.nocoDb`  
  Reads all existing rows from the `EOLDates` table.
- **Configuration choices:**  
  - Operation: `getAll`
  - `returnAll: true`
  - `alwaysOutputData: true`
  - Table ID `mmk9jtv13jor4jh`
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: `Each Software Once`
  - Output: `Make ID Key`
- **Version-specific requirements:** NocoDB node v3.
- **Edge cases / failures:**  
  - Large table reads every iteration are inefficient.
  - Auth/project/table mismatch.
  - If table is empty, merge logic still works because `alwaysOutputData` keeps the branch alive.
- **Sub-workflow reference:** None.

#### Make ID Key
- **Type and role:** `n8n-nodes-base.set`  
  Shapes existing EOLDates rows so they can be merged against incoming API records.
- **Configuration choices:** Sets:
  - `Id`
  - `key`
  - `Projects` from `_nc_m2m_EOLDates With P_EOLProjects`
- **Key expressions or variables used:**  
  - `{{ $json.Id }}`
  - `{{ $json.key }}`
  - `{{ $json["_nc_m2m_EOLDates With P_EOLProjects"] }}`
- **Input / output connections:**  
  - Input: `Get Current Rows`
  - Output: `Merge` (input 1)
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**  
  - Relation field internal name may change between NocoDB versions or schemas.
  - Missing `key` breaks merge matching.
- **Sub-workflow reference:** None.

#### Merge
- **Type and role:** `n8n-nodes-base.merge`  
  Enriches incoming API records with existing database record information based on `key`.
- **Configuration choices:**  
  - Mode: `combine`
  - Join mode: `enrichInput1`
  - Match field: `key`
- **Key expressions or variables used:** Uses `key` field from both inputs.
- **Input / output connections:**  
  - Input 0: `Rename Keys`
  - Input 1: `Make ID Key`
  - Output: `Convert bool to null`
- **Version-specific requirements:** Merge v3.
- **Edge cases / failures:**  
  - Non-unique keys produce ambiguous merges.
  - Missing `key` on either side prevents enrichment.
- **Sub-workflow reference:** None.

#### Convert bool to null
- **Type and role:** `n8n-nodes-base.code`  
  Converts boolean values in date fields to `null`.
- **Configuration choices:** JavaScript loops over all input items and replaces:
  - boolean `End Of Life` → `null`
  - boolean `End Of Support` → `null`
- **Key expressions or variables used:** Uses `$input.all()`.
- **Input / output connections:**  
  - Input: `Merge`
  - Output: `If ID exists`
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**  
  - Does not convert `Subversion Release` booleans.
  - If endoflife.date returns unexpected types, comparisons may still fail later.
- **Sub-workflow reference:** None.

#### If ID exists
- **Type and role:** `n8n-nodes-base.if`  
  Decides whether to update an existing NocoDB row or insert a new one.
- **Configuration choices:**  
  - Condition: `Id > 0`
- **Key expressions or variables used:**  
  - `{{ $json.Id }}`
- **Input / output connections:**  
  - Input: `Convert bool to null`
  - True output: `Update Data`
  - False output: `Insert New`
- **Version-specific requirements:** If node v2.2.
- **Edge cases / failures:**  
  - If `Id` is missing or string-typed unexpectedly, strict validation may cause mismatches.
- **Sub-workflow reference:** None.

#### Update Data
- **Type and role:** `n8n-nodes-base.nocoDb`  
  Updates an existing `EOLDates` row.
- **Configuration choices:**  
  - Operation: `update`
  - Data mode: `autoMapInputData`
- **Key expressions or variables used:** Uses incoming JSON fields automatically.
- **Input / output connections:**  
  - Input: `If ID exists` true branch
  - Output: `Wait`
- **Version-specific requirements:** NocoDB node v3.
- **Edge cases / failures:**  
  - Requires the correct row identifier to be present.
  - Auto-mapping may include fields NocoDB doesn’t accept.
- **Sub-workflow reference:** None.

#### Insert New
- **Type and role:** `n8n-nodes-base.nocoDb`  
  Creates a new `EOLDates` row when no matching key exists.
- **Configuration choices:**  
  - Operation: `create`
  - Data mode: `autoMapInputData`
- **Key expressions or variables used:** Auto-maps incoming JSON.
- **Input / output connections:**  
  - Input: `If ID exists` false branch
  - Output: `Wait`
- **Version-specific requirements:** NocoDB node v3.
- **Edge cases / failures:**  
  - If `key` is not unique, duplicate rows can accumulate.
  - Auto-mapping extra fields may be ignored or rejected depending on NocoDB config.
- **Sub-workflow reference:** None.

#### Wait
- **Type and role:** `n8n-nodes-base.wait`  
  Pauses before the loop continues, used as a throttle mechanism.
- **Configuration choices:** No explicit wait parameters are defined in the export.
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: `Update Data`, `Insert New`
  - Output: `Each Software Once`
- **Version-specific requirements:** Wait node v1.1.
- **Edge cases / failures:**  
  - Because no explicit delay settings are visible, actual behavior depends on node defaults and n8n version/runtime behavior.
- **Sub-workflow reference:** None.

---

## 2.6 Project Evaluation

**Overview:**  
After the software refresh loop completes, this block loads projects and their linked software rows, then classifies those rows into alert buckets.

**Nodes Involved:**  
- Get EOLProjects
- Loop over projects
- Get many rows
- Combine All Software into single list
- Merge Project with Softwares it uses
- Check when is EOL

### Node Details

#### Get EOLProjects
- **Type and role:** `n8n-nodes-base.nocoDb`  
  Loads all project records from `EOLProjects`.
- **Configuration choices:**  
  - Operation: `getAll`
  - `returnAll: true`
  - `executeOnce: true`
  - Table ID `m68cljurlid8t2z`
- **Key expressions or variables used:** None.
- **Input / output connections:**  
  - Input: `Each Software Once`
  - Output: `Loop over projects`
- **Version-specific requirements:** NocoDB node v3.
- **Edge cases / failures:**  
  - This placement means project evaluation begins after the software loop branch reaches completion behavior.
  - Empty projects table means no alerts.
- **Sub-workflow reference:** None.

#### Loop over projects
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates through projects one at a time.
- **Configuration choices:** Default batch loop settings.
- **Key expressions or variables used:** Downstream uses current project `Id`.
- **Input / output connections:**  
  - Input: `Get EOLProjects`
  - Output 1: `Get many rows`, `Merge Project with Softwares it uses`
  - Loop-back input from `Send a message`
- **Version-specific requirements:** Split In Batches v3.
- **Edge cases / failures:**  
  - If a project has no linked software, later nodes may produce empty arrays.
- **Sub-workflow reference:** None.

#### Get many rows
- **Type and role:** `n8n-nodes-base.nocoDb`  
  Fetches all `EOLDates` rows linked to the current project.
- **Configuration choices:**  
  - Operation: `getAll`
  - `returnAll: true`
  - `executeOnce: true`
  - Table: `EOLDates`
  - Where clause filters on `EOLProjects_id = current project Id`
- **Key expressions or variables used:**  
  - `=(EOLProjects_id,eq,{{ $json.Id }})`
- **Input / output connections:**  
  - Input: `Loop over projects`
  - Output: `Combine All Software into single list`
- **Version-specific requirements:** NocoDB node v3.
- **Edge cases / failures:**  
  - Relation field naming may vary by NocoDB schema.
  - If no linked rows exist, the next node gets an empty input set.
- **Sub-workflow reference:** None.

#### Combine All Software into single list
- **Type and role:** `n8n-nodes-base.set`  
  Wraps all fetched software rows for a project into one array field named `EOLDates`.
- **Configuration choices:**  
  - Sets `EOLDates = $input.all().map(i => i.json)`
- **Key expressions or variables used:**  
  - `{{ $input.all().map(i=>i.json) }}`
- **Input / output connections:**  
  - Input: `Get many rows`
  - Output: `Merge Project with Softwares it uses` (input 1)
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**  
  - Empty input yields an empty array, which is acceptable.
- **Sub-workflow reference:** None.

#### Merge Project with Softwares it uses
- **Type and role:** `n8n-nodes-base.merge`  
  Merges current project metadata with the consolidated list of linked EOLDates rows.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineByPosition`
- **Key expressions or variables used:** Positional merge only.
- **Input / output connections:**  
  - Input 0: `Loop over projects`
  - Input 1: `Combine All Software into single list`
  - Output: `Check when is EOL`
- **Version-specific requirements:** Merge v3.2.
- **Edge cases / failures:**  
  - Positional combine assumes exactly one project item and one aggregated software-list item per iteration.
- **Sub-workflow reference:** None.

#### Check when is EOL
- **Type and role:** `n8n-nodes-base.code`  
  Computes three arrays on each project:
  - `EOL_Today`
  - `Past_EOL`
  - `EOL_in_x_days`
- **Configuration choices:**  
  - Uses current UTC date string `YYYY-MM-DD`
  - Reads `Days before EOL` from `Config`, default fallback `30`
  - Operates on `item.json.EOLDates`
  - Removes `_nc_m2m_EOLDates_EOLProjects`
- **Key expressions or variables used:**  
  - `$now.toUTC().toString().slice(0, 10)`
  - `$('Config').first().json["Days before EOL"] ?? 30`
  - `DateTime.fromJSDate(...)`
- **Input / output connections:**  
  - Input: `Merge Project with Softwares it uses`
  - Output: `Filter`
- **Version-specific requirements:** Code node v2.  
  Relies on Luxon-style `DateTime` availability in the code runtime.
- **Edge cases / failures:**  
  - The logic for `EOL_in_x_days` uses `$now.diff(targetDate, 'days').days <= daysBeforeEol`; because past dates produce negative values, exclusion sets are essential.
  - Invalid date strings may become `Invalid Date`.
  - Null values converted earlier can still become invalid `Date` objects.
  - `Subversion Release` was not sanitized if boolean.
- **Sub-workflow reference:** None.

---

## 2.7 Filtering and Slack Notification

**Overview:**  
This block keeps only projects with at least one EOL issue and sends a richly formatted Slack message summarizing the affected software grouped by severity.

**Nodes Involved:**  
- Filter
- Send a message

### Node Details

#### Filter
- **Type and role:** `n8n-nodes-base.filter`  
  Discards unaffected projects.
- **Configuration choices:**  
  - OR-combined conditions:
    - `Past_EOL` array not empty
    - `EOL_in_x_days` array not empty
    - `EOL_Today` array not empty
- **Key expressions or variables used:**  
  - `{{ $json.Past_EOL }}`
  - `{{ $json.EOL_in_x_days }}`
  - `{{ $json.EOL_Today }}`
- **Input / output connections:**  
  - Input: `Check when is EOL`
  - Output: `Send a message`
- **Version-specific requirements:** Filter node v2.3.
- **Edge cases / failures:**  
  - If those properties are missing or not arrays, strict validation may fail.
- **Sub-workflow reference:** None.

#### Send a message
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a block-based Slack message to a fixed channel.
- **Configuration choices:**  
  - Auth: Slack OAuth2
  - Message type: `block`
  - Target channel ID: `C0AM2GZ70SG`
  - Dynamically builds Slack Block Kit JSON with sections for:
    - header containing `Project Name`
    - `Past_EOL`
    - `EOL_Today`
    - `EOL_in_x_days`
  - Includes software name, version, end-of-life date, end-of-support date, and subversion release date.
- **Key expressions or variables used:**  
  - `{{ $json['Project Name'] }}`
  - `.map(...)` over `Past_EOL`, `EOL_Today`, `EOL_in_x_days`
  - `$if(...)` conditional block insertion
- **Input / output connections:**  
  - Input: `Filter`
  - Output: `Loop over projects` (loop continuation)
- **Version-specific requirements:** Slack node v2.4.
- **Edge cases / failures:**  
  - Invalid block JSON due to expression rendering or trailing commas.
  - Missing arrays may break `.length` or `.map`.
  - Slack channel permissions/auth issues.
  - Message size may exceed Slack limits if many software items are affected.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Create Tables | Manual Trigger | Manual bootstrap entry point for NocoDB schema creation |  | NocoDB Config | ## Before you start:<br>This workflow requires 3 NocoDB Tables which can be created with single click by pressing "Create Tables". It will create required tables. It's important to then populate EOL Software table by hand, run workflow again manually, this time from "Run daily" trigger and populate EOL Projects table using values from EOLDates. |
| NocoDB Config | Set | Stores Base ID and NocoDB URL for metadata API calls | Create Tables | Create NocoDB EOL Software Table | ## Before you start:<br>This workflow requires 3 NocoDB Tables which can be created with single click by pressing "Create Tables". It will create required tables. It's important to then populate EOL Software table by hand, run workflow again manually, this time from "Run daily" trigger and populate EOL Projects table using values from EOLDates. |
| Create NocoDB EOL Software Table | HTTP Request | Creates EOLSoftware table in NocoDB | NocoDB Config | Create NocoDB EOL Dates Table | ## Before you start:<br>This workflow requires 3 NocoDB Tables which can be created with single click by pressing "Create Tables". It will create required tables. It's important to then populate EOL Software table by hand, run workflow again manually, this time from "Run daily" trigger and populate EOL Projects table using values from EOLDates. |
| Create NocoDB EOL Dates Table | HTTP Request | Creates EOLDates table in NocoDB | Create NocoDB EOL Software Table | Create NocoDB EOL Projects Table | ## Before you start:<br>This workflow requires 3 NocoDB Tables which can be created with single click by pressing "Create Tables". It will create required tables. It's important to then populate EOL Software table by hand, run workflow again manually, this time from "Run daily" trigger and populate EOL Projects table using values from EOLDates. |
| Create NocoDB EOL Projects Table | HTTP Request | Creates EOLProjects table in NocoDB | Create NocoDB EOL Dates Table | Create relation between tables | ## Before you start:<br>This workflow requires 3 NocoDB Tables which can be created with single click by pressing "Create Tables". It will create required tables. It's important to then populate EOL Software table by hand, run workflow again manually, this time from "Run daily" trigger and populate EOL Projects table using values from EOLDates. |
| Create relation between tables | HTTP Request | Creates link field from projects to EOLDates | Create NocoDB EOL Projects Table |  | ## Before you start:<br>This workflow requires 3 NocoDB Tables which can be created with single click by pressing "Create Tables". It will create required tables. It's important to then populate EOL Software table by hand, run workflow again manually, this time from "Run daily" trigger and populate EOL Projects table using values from EOLDates. |
| Run daily | Schedule Trigger | Main scheduled entry point |  | Config | ## Fill out config<br>How many days before End of life date, End of support date, or Subversion Release should you be notified  |
| Config | Set | Stores notification threshold in days | Run daily | Get software to check | ## Fill out config<br>How many days before End of life date, End of support date, or Subversion Release should you be notified  |
| Get software to check | NocoDB | Reads tracked software list from EOLSoftware | Config | Each Software Once | ## Step 1: Fetch softwares you are using from NocoDB EOLSoftware Table and feed them to into pipeline<br>Loops are perfect tool if we want to process each item individaully with more than one block and retrun combined result or like in this case you want to better visually represent ongoing process. |
| Each Software Once | Split In Batches | Iterates over software entries | Get software to check, Wait | Get EOLProjects, Get Current Rows, Get EOL data from endoflife.date | ## Step 1: Fetch softwares you are using from NocoDB EOLSoftware Table and feed them to into pipeline<br>Loops are perfect tool if we want to process each item individaully with more than one block and retrun combined result or like in this case you want to better visually represent ongoing process. |
| Get EOL data from endoflife.date | HTTP Request | Fetches lifecycle data from endoflife.date API | Each Software Once | Add Key and Software | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| Add Key and Software | Set | Adds software name and synthetic merge key to API rows | Get EOL data from endoflife.date | Rename Keys | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| Rename Keys | Rename Keys | Renames API fields to NocoDB-friendly labels | Add Key and Software | Merge | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| Get Current Rows | NocoDB | Loads existing EOLDates rows for sync comparison | Each Software Once | Make ID Key | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| Make ID Key | Set | Extracts ID, key, and linked projects from existing rows | Get Current Rows | Merge | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| Merge | Merge | Matches new API rows with existing EOLDates rows by key | Rename Keys, Make ID Key | Convert bool to null | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| Convert bool to null | Code | Normalizes boolean date fields to null | Merge | If ID exists | ## Step 3: Fetch End of life dates from endoflife.date. Merge new data with existing one<br>https://endoflife.date offers End of life data for over 400 products. We can access this data using their free API. We rename keys to more human readable form, add some metadata and we convert dates from `false` value to `null` using [code block](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/). In this step we are using EOLDates Noco Table to check if we already have data regarding software, if so, we update it, if not, we add it to our database |
| If ID exists | If | Decides update vs insert in EOLDates | Convert bool to null | Update Data, Insert New | ## Step 4: Update EOL Dates table with newest info<br>We check if given software already has entry in the database, update it if it does, or insert new one if it doesn't. After that we wait to make sure we don't hit rate limits of endoflife.data |
| Update Data | NocoDB | Updates existing EOLDates record | If ID exists | Wait | ## Step 4: Update EOL Dates table with newest info<br>We check if given software already has entry in the database, update it if it does, or insert new one if it doesn't. After that we wait to make sure we don't hit rate limits of endoflife.data |
| Insert New | NocoDB | Creates new EOLDates record | If ID exists | Wait | ## Step 4: Update EOL Dates table with newest info<br>We check if given software already has entry in the database, update it if it does, or insert new one if it doesn't. After that we wait to make sure we don't hit rate limits of endoflife.data |
| Wait | Wait | Throttles loop iterations before continuing | Update Data, Insert New | Each Software Once | ## Step 4: Update EOL Dates table with newest info<br>We check if given software already has entry in the database, update it if it does, or insert new one if it doesn't. After that we wait to make sure we don't hit rate limits of endoflife.data |
| Get EOLProjects | NocoDB | Loads all projects to evaluate for alerts | Each Software Once | Loop over projects | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Loop over projects | Split In Batches | Iterates over each project | Get EOLProjects, Send a message | Get many rows, Merge Project with Softwares it uses | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Get many rows | NocoDB | Fetches EOLDates rows linked to current project | Loop over projects | Combine All Software into single list | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Combine All Software into single list | Set | Aggregates all linked software rows into one EOLDates array | Get many rows | Merge Project with Softwares it uses | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Merge Project with Softwares it uses | Merge | Joins project data with its aggregated EOLDates list | Loop over projects, Combine All Software into single list | Check when is EOL | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Check when is EOL | Code | Classifies linked software into past/today/incoming EOL arrays | Merge Project with Softwares it uses | Filter | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Filter | Filter | Keeps only projects with at least one alert-worthy EOL event | Check when is EOL | Send a message | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Send a message | Slack | Sends formatted Slack EOL alert for one affected project | Filter | Loop over projects | ## Step 5: Loop over EOLProjects table records, check if any has EOL date comming and notify the user on the Slack channel<br>We use code block to check if we are Past EOL, is it soon or it's today. Code block allows us to do it all at once in more cohesieve and easier way than using multiple other blocks. We filter projects that are not affected (EOL date is too far into future). At the end we send a message including project name with all events properly structured to predefined Slack channel. |
| Sticky Note | Sticky Note | Documentation note |  |  |  |
| Sticky Note1 | Sticky Note | Documentation note |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note |  |  |  |
| Sticky Note4 | Sticky Note | Documentation note |  |  |  |
| Sticky Note5 | Sticky Note | Documentation note |  |  |  |
| Sticky Note8 | Sticky Note | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites
1. Prepare an n8n instance.
2. Prepare a NocoDB instance and generate a **NocoDB API Token** credential in n8n.
3. Prepare a Slack app / OAuth2 credential with permission to post to the desired channel.
4. Know your NocoDB:
   - base ID
   - base/project URL
   - final table IDs after creation if you plan to use the NocoDB node exactly like this workflow

## Build the setup path

1. **Add a Manual Trigger** node named **Create Tables**.

2. **Add a Set** node named **NocoDB Config** after it.
   - Add string field `Base ID`
   - Add string field `NocoDB Url`
   - Fill both manually
   - Example URL format should be your NocoDB base URL prefix used for metadata API calls, such as `https://your-nocodb-instance/`

3. **Add an HTTP Request** node named **Create NocoDB EOL Software Table**.
   - Method: `POST`
   - URL:
     `={{ $json["NocoDB Url"] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables`
   - Authentication: predefined credential type
   - Credential type: `NocoDB API Token`
   - Send body: JSON
   - Body creates table `EOLSoftware` with one field:
     - `Software` / `SingleLineText`

4. **Add an HTTP Request** node named **Create NocoDB EOL Dates Table**.
   - Same auth style
   - URL:
     `={{ $('NocoDB Config').first().json['NocoDB Url'] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables`
   - JSON body creates table `EOLDates` with fields:
     - `key`
     - `Software`
     - `Version`
     - `Latest Subversion`
     - `Subversion Release`
     - `End Of Life`
     - `End Of Support`
   - All as `SingleLineText`

5. **Add an HTTP Request** node named **Create NocoDB EOL Projects Table**.
   - Same auth style
   - URL:
     `={{ $('NocoDB Config').first().json['NocoDB Url'] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables`
   - JSON body creates table `EOLProjects` with:
     - `Project Name` / `SingleLineText`

6. **Add an HTTP Request** node named **Create relation between tables**.
   - Method: `POST`
   - URL:
     `={{ $('NocoDB Config').first().json['NocoDB Url'] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables/{{ $json.id }}/fields`
   - Authentication: predefined credential type
   - Body defines:
     - title: `EOLDates`
     - type: `Links`
     - relation_type: `hm`
     - related_table_id from **Create NocoDB EOL Dates Table** output

7. Connect setup nodes in this order:
   - `Create Tables` → `NocoDB Config`
   - `NocoDB Config` → `Create NocoDB EOL Software Table`
   - `Create NocoDB EOL Software Table` → `Create NocoDB EOL Dates Table`
   - `Create NocoDB EOL Dates Table` → `Create NocoDB EOL Projects Table`
   - `Create NocoDB EOL Projects Table` → `Create relation between tables`

8. Run **Create Tables** once.

9. In NocoDB, manually populate `EOLSoftware` with software slugs matching `https://endoflife.date` product names, for example `nodejs`, `ubuntu`, `python`, etc.

## Build the scheduled runtime path

10. **Add a Schedule Trigger** node named **Run daily**.
    - Configure it to run every day at hour `7`.

11. **Add a Set** node named **Config** after it.
    - Add number field `Days before EOL`
    - Set default value to `31`

12. **Add a NocoDB** node named **Get software to check**.
    - Credential: `NocoDB API Token`
    - Operation: `getAll`
    - Select your NocoDB base/project
    - Table: `EOLSoftware`
    - `Return All = true`

13. **Add a Split In Batches** node named **Each Software Once**.
    - Use default settings.

14. Connect:
    - `Run daily` → `Config`
    - `Config` → `Get software to check`
    - `Get software to check` → `Each Software Once`

## Build the EOLDates refresh branch

15. **Add an HTTP Request** node named **Get EOL data from endoflife.date**.
    - Method: `GET`
    - URL:
      `=https://endoflife.date/api/{{ $json.Software.toLowerCase() }}.json`

16. **Add a Set** node named **Add Key and Software**.
    - Keep other fields enabled
    - Add string field `key`:
      `={{ $("Each Software Once").item.json.Software }}{{ $json.cycle }}`
    - Add string field `Software`:
      `={{ $("Each Software Once").item.json.Software }}`

17. **Add a Rename Keys** node named **Rename Keys**.
    - Configure these renames:
      - `cycle` → `Version`
      - `latest` → `Latest Subversion`
      - `latestReleaseDate` → `Subversion Release`
      - `eol` → `End Of Life`
      - `lts` → `Long Term Support`
      - `support` → `End Of Support`

18. **Add a NocoDB** node named **Get Current Rows**.
    - Operation: `getAll`
    - Table: `EOLDates`
    - `Return All = true`
    - `Always Output Data = true`

19. **Add a Set** node named **Make ID Key**.
    - Add number field `Id`:
      `={{ $json.Id }}`
    - Add string field `key`:
      `={{ $json.key }}`
    - Add array field `Projects`:
      `={{ $json["_nc_m2m_EOLDates With P_EOLProjects"] }}`
    - Note: the relation field name may differ in your NocoDB instance. Adjust accordingly.

20. **Add a Merge** node named **Merge**.
    - Mode: `Combine`
    - Join mode: `Enrich Input 1`
    - Match field: `key`

21. **Add a Code** node named **Convert bool to null**.
    - Use this logic:
      - Loop over all items
      - If `End Of Life` is boolean, set it to `null`
      - If `End Of Support` is boolean, set it to `null`
      - Return all items

22. **Add an If** node named **If ID exists**.
    - Condition: number `Id > 0`

23. **Add a NocoDB** node named **Update Data**.
    - Operation: `update`
    - Table: `EOLDates`
    - Data to send: `autoMapInputData`

24. **Add a NocoDB** node named **Insert New**.
    - Operation: `create`
    - Table: `EOLDates`
    - Data to send: `autoMapInputData`

25. **Add a Wait** node named **Wait**.
    - Keep a short pause if desired to throttle requests.
    - Since the exported workflow shows no explicit parameters, set a small delay manually if you want predictable behavior.

26. Connect the refresh branch:
    - `Each Software Once` → `Get EOL data from endoflife.date`
    - `Get EOL data from endoflife.date` → `Add Key and Software`
    - `Add Key and Software` → `Rename Keys`
    - `Each Software Once` → `Get Current Rows`
    - `Get Current Rows` → `Make ID Key`
    - `Rename Keys` → `Merge` input 1
    - `Make ID Key` → `Merge` input 2
    - `Merge` → `Convert bool to null`
    - `Convert bool to null` → `If ID exists`
    - `If ID exists` true → `Update Data`
    - `If ID exists` false → `Insert New`
    - `Update Data` → `Wait`
    - `Insert New` → `Wait`
    - `Wait` → `Each Software Once` to continue the loop

## Build the project alert branch

27. **Add a NocoDB** node named **Get EOLProjects**.
    - Operation: `getAll`
    - Table: `EOLProjects`
    - `Return All = true`
    - `Execute Once = true`

28. Connect `Each Software Once` to **Get EOLProjects**.
   - This branch should execute after the software loop completes in the Split In Batches flow pattern.

29. **Add a Split In Batches** node named **Loop over projects**.

30. Connect `Get EOLProjects` → `Loop over projects`.

31. **Add a NocoDB** node named **Get many rows**.
    - Operation: `getAll`
    - Table: `EOLDates`
    - `Return All = true`
    - Add a where clause:
      `=(EOLProjects_id,eq,{{ $json.Id }})`
    - If your relation field is named differently, adjust the where condition.

32. **Add a Set** node named **Combine All Software into single list**.
    - Add array field `EOLDates`:
      `={{ $input.all().map(i=>i.json) }}`

33. **Add a Merge** node named **Merge Project with Softwares it uses**.
    - Mode: `Combine`
    - Combine by: `Position`

34. Connect:
    - `Loop over projects` second output → `Get many rows`
    - `Get many rows` → `Combine All Software into single list`
    - `Loop over projects` second output → `Merge Project with Softwares it uses` input 1
    - `Combine All Software into single list` → `Merge Project with Softwares it uses` input 2

35. **Add a Code** node named **Check when is EOL**.
    - Paste logic equivalent to:
      - derive today's UTC date in `YYYY-MM-DD`
      - read `Days before EOL` from `Config`
      - build `EOL_Today`
      - build `Past_EOL` excluding today's items
      - build `EOL_in_x_days` excluding today's and past items
      - remove `_nc_m2m_EOLDates_EOLProjects`
    - Ensure Luxon `DateTime` is available in your n8n code environment if using the same expressions.

36. **Add a Filter** node named **Filter**.
    - OR logic
    - Keep items where any of these arrays is not empty:
      - `Past_EOL`
      - `EOL_in_x_days`
      - `EOL_Today`

37. **Add a Slack** node named **Send a message**.
    - Credential: Slack OAuth2
    - Select: channel
    - Channel: choose your target channel
    - Message type: `block`
    - Build a block JSON payload that:
      - prints `Project Name`
      - shows sections for `Past_EOL`, `EOL_Today`, `EOL_in_x_days`
      - loops over arrays with expressions
    - You can reuse the same block structure from the exported workflow.

38. Connect:
    - `Merge Project with Softwares it uses` → `Check when is EOL`
    - `Check when is EOL` → `Filter`
    - `Filter` → `Send a message`
    - `Send a message` → `Loop over projects` to continue the project loop

## Final manual data setup

39. After the setup path has created the tables:
    - Add rows to `EOLSoftware` manually with valid endoflife.date slugs.
    - Run the schedule path once manually to populate `EOLDates`.
    - Then add rows to `EOLProjects`.
    - Link each project to relevant `EOLDates` records through the created relation field.

## Credential configuration summary

40. **NocoDB credential**
    - Use `NocoDB API Token`
    - Required by:
      - Get software to check
      - Get Current Rows
      - Update Data
      - Insert New
      - Get EOLProjects
      - Get many rows
      - all setup HTTP metadata calls

41. **Slack credential**
    - Use Slack OAuth2
    - Ensure channel posting scopes are granted
    - Verify the selected channel is accessible to the app/bot

## Important rebuild constraints

42. The software values in `EOLSoftware.Software` must match endoflife.date slugs exactly.

43. The synthetic `key` should remain stable. If you change its formula, old rows in `EOLDates` will no longer match for updates.

44. If your NocoDB-generated relation field names differ from those in the expressions:
    - update `Make ID Key`
    - update `Get many rows` where clause
    - update cleanup code in `Check when is EOL`

45. Consider explicitly configuring the **Wait** node delay, because the exported workflow does not expose a concrete wait duration.

46. Consider extending the boolean-to-null conversion to include `Subversion Release` as well if endoflife.date can return `false` there.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| End of life alert: This workflow monitors your projects for software that is approaching or has passed its End of Life date, by fetching up-to-date data from `https://endoflife.date` and cross-referencing it with software versions used in each of your projects stored in NocoDB. When an issue is detected, a structured alert is sent to a designated Slack channel. | Overview |
| Setup guidance: Run the Create Tables trigger once to provision tables, populate `EOLSoftware`, run the daily path to populate `EOLDates`, then populate `EOLProjects` and link projects to software versions. | Operational setup |
| Customize guidance: Slack channel and message structure can be changed directly in the `Send a message` node. | Slack customization |
| endoflife.date public API | https://endoflife.date |
| n8n Code node documentation | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/ |
| Help / contact | [developers@sailingbyte.com](mailto:developers@sailingbyte.com) |
| Project site | [sailingbyte.com](sailingbyte.com) |

