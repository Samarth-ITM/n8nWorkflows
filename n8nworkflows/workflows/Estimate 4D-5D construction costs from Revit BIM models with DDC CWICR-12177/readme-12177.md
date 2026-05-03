Estimate 4D/5D construction costs from Revit BIM models with DDC CWICR

https://n8nworkflows.xyz/workflows/estimate-4d-5d-construction-costs-from-revit-bim-models-with-ddc-cwicr-12177


# Estimate 4D/5D construction costs from Revit BIM models with DDC CWICR

## 1. Workflow Overview

**Title (given):** *Estimate 4D/5D construction costs from Revit BIM models with DDC CWICR*  
**Workflow name (JSON):** *V3.1_CAD_(BIM)_Cost_Estimation_Pipeline_with_DDC_CWICR ES copy*

This workflow automates **cost estimation from a Revit (.rvt) BIM model** by:
1) exporting BIM quantities to Excel,  
2) grouping and filtering out non-physical elements,  
3) using LLMs to infer project context and decompose element types into work items,  
4) retrieving matching rate positions from a **Qdrant vector database** (DDC CWICR dataset),  
5) mapping quantities/units and calculating costs/resources,  
6) aggregating by type and by construction phase,  
7) producing an **HTML + Excel-compatible XLS** report saved next to the Revit model.

### 1.1 Logical Blocks (by direct dependencies)

**Block 0 — Entry & configuration**
- Manual trigger + project file paths + language/vector DB selection.

**Block 1 — Conversion**
- Check whether the Excel export already exists; if not, run an external Revit exporter (`RvtExporter.exe`).

**Block 2 — Data loading & grouping + element filtering**
- Read the exported Excel, filter “standard 3D view” elements, let AI decide aggregation rules, group by “Type Name”, classify categories as building/non-building, hard-exclude known non-physical categories.

**Block 3 — Project analysis (Stages 0–3)**
- Stage 0: build a deduplicated “type list” with aggregated quantities.
- Stage 1: detect project type/scale.
- Stage 2: generate construction phases.
- Stage 2.5: compact type list for token reduction.
- Stage 3: assign types to phases (LLM), then re-enrich using all BIM types.

**Block 4 — Work decomposition loop (Stage 4)**
- Iterate each type → LLM generates work items (minimum 2).
- Parse/validate work list; generate fallbacks; prepare per-work items for pricing.

**Block 5 — Pricing & calculation (Stages 5–7)**
- Prepare search strategies → generate embeddings → Qdrant vector search → parse best rate → map BIM quantity to rate unit → compute costs/resources.

**Block 6 — Validation & aggregation (Stages 7.5–8)**
- Aggregate works per type; optional LLM validation; store per-type results; final aggregation by phases with **category-based correction**.

**Block 7 — Reporting (Stage 9)**
- Generate HTML/XLS, save to project folder, open in browser (Windows).

---

## 2. Block-by-Block Analysis

### Block 0 — Entry & Configuration

**Overview:** Initializes workflow inputs: file paths, grouping key, and language selection which drives currency, locale, and Qdrant collection.

**Nodes involved**
- When clicking 'Execute workflow'
- Setup - Define file paths1
- Configure Language & Vector DB
- Create - Excel filename1

#### Node: **When clicking 'Execute workflow'**
- **Type/role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – workflow entry point.
- **Config:** No parameters.
- **Outputs:** Single empty item triggers config chain.
- **Failure cases:** None.

#### Node: **Setup - Define file paths1**
- **Type/role:** Set node – defines key user inputs.
- **Key fields:**
  - `path_to_converter`: Windows path to `RvtExporter.exe`
  - `project_file`: path to `.rvt`
  - `group_by`: “Type Name”
  - `user_project_description`: optional
  - `language_code`: e.g., `DE`, `EN`, `ES`, etc.
- **Downstream usage:** consumed by Configure Language & Vector DB and later Stage 9 report naming.
- **Failure cases:** incorrect paths will break conversion / saving.
- **Version notes:** Set node v3.4.

#### Node: **Configure Language & Vector DB**
- **Type/role:** Code node – maps `language_code` to:
  - city/pricing level
  - Qdrant collection name
  - currency & locale
  - system prompt language instruction (`system_prompt_lang`)
  - `search_lang`
- **Outputs:** adds convenience fields and `pricing_standards`.
- **Failure cases:** invalid `language_code` defaults to DE; missing upstream fields propagate null paths.
- **Version notes:** Code node v2.

#### Node: **Create - Excel filename1**
- **Type/role:** Set node – derives Excel file name from `project_file`.
- **Key expression:** builds `xlsx_filename` as: “<project path without extension>_<original extension>.xlsx”.
- **Pass-through fields:** language/qdrant/currency/system prompt fields.
- **Failure cases:** if `project_file` is empty, string ops may produce invalid path.

---

### Block 1 — Conversion Block

**Overview:** Avoids re-exporting if an Excel file already exists; otherwise runs an external converter via Execute Command.

**Nodes involved**
- Check - Does Excel file exist?1
- If - File exists?1
- Info - Skip conversion1
- Extract - Run converter1
- Check - Did extraction succeed?1
- Error - Show what went wrong1
- Set xlsx_filename after success1
- Merge - Continue workflow1
- Set Parameters1

#### Node: **Check - Does Excel file exist?1**
- **Type/role:** Read Binary File – attempts to read `xlsx_filename` to detect existence.
- **Config:** `continueOnFail: true`, `alwaysOutputData: true` (critical: workflow continues even if missing).
- **Logic:** if file exists, `$binary.data` present.
- **Failure cases:** permissions, path invalid; handled via `continueOnFail`.

#### Node: **If - File exists?1**
- **Type/role:** IF node – checks `{{ $binary.data ? true : false }}` equals true.
- **Outputs:**
  - **True branch:** Info - Skip conversion1
  - **False branch:** Extract - Run converter1
- **Failure cases:** if upstream outputs a binary object for unrelated reasons, may falsely “exist”.

#### Node: **Info - Skip conversion1**
- **Type/role:** Set node – emits status and `xlsx_filename`.

#### Node: **Extract - Run converter1**
- **Type/role:** Execute Command – runs converter with args:
  - `"path_to_converter" "project_file"`
- **Config:** `continueOnFail: true` (errors are captured instead of stopping immediately).
- **Critical environment requirement (n8n 2.0+):** Execute Command is excluded by default; requires `NODES_EXCLUDE=[]` (see sticky note).
- **Failure cases:** Windows quoting, missing exe, Revit/version mismatch, non-zero exit code.

#### Node: **Check - Did extraction succeed?1**
- **Type/role:** IF – checks whether `Extract - Run converter1`.json.error exists.
- **Note:** The condition uses “object exists”; this can be counterintuitive—ensure it routes correctly in n8n.
- **Outputs:** error path vs success path.

#### Node: **Error - Show what went wrong1**
- **Type/role:** Set node – builds an error summary from converter node output.
- **Failure cases:** error fields missing; falls back to “Unknown error”.

#### Node: **Set xlsx_filename after success1**
- **Type/role:** Set node – propagates the filename for downstream.

#### Node: **Merge - Continue workflow1**
- **Type/role:** Merge node – merges the “skipped” path and “success/error” path to continue.
- **Failure cases:** incorrect merge mode expectations (defaults); ensure it produces one consistent item.

#### Node: **Set Parameters1**
- **Type/role:** Set node – standardizes `path_to_file = xlsx_filename`.
- **Downstream:** Read Excel File1 uses `path_to_file`.

---

### Block 2 — Data Loading & AI Classification

**Overview:** Loads the exported Excel rows, filters elements to those visible in the standard 3D view, uses AI to infer aggregation rules, groups by type, and classifies categories into building vs non-building (with a hard exclude list).

**Nodes involved**
- Read Excel File1
- Parse Excel1
- On the standard 3D View
- Non-3D View Elements Output1
- Extract Headers and Data
- CONFIG - AI Headers
- AI Analyze All Headers
- Process AI Response1
- Group Data with AI Rules1
- Find Category Fields
- CONFIG - AI Classify
- AI Classify Categories
- Apply Classification to Groups
- Is Building Element1
- Non-Building Elements Output1

#### Node: **Read Excel File1**
- **Type/role:** Read Binary File – loads Excel at `{{$json.path_to_file}}`.
- **Failure cases:** file not found, permission.

#### Node: **Parse Excel1**
- **Type/role:** Spreadsheet File – converts XLSX binary to JSON rows.
- **Key config:**
  - `headerRow: true`
  - `sheetName`: from `Set Parameters1`.json.sheet_name (note: `sheet_name` is not set elsewhere in the provided workflow; default behavior must be validated).
- **Failure cases:** wrong sheet name, malformed xlsx, large sheets memory.

#### Node: **On the standard 3D View**
- **Type/role:** IF node – filters rows where `On the standard 3D View : Boolean` is true.
- **Outputs:**
  - True → Extract Headers and Data
  - False → Non-3D View Elements Output1
- **Failure cases:** column name mismatch (localization/export differences).

#### Node: **Non-3D View Elements Output1**
- **Type/role:** Set node – emits count of filtered-out elements.

#### Node: **Extract Headers and Data**
- **Type/role:** Code – computes:
  - unique headers across rows
  - header cleanup (removes `: string/double/...` suffixes)
  - `headerMapping` old→clean
  - `sampleValues`
  - stores `rawData`
- **Failure cases:** no items throws error.

#### Node: **CONFIG - AI Headers**
- **Type/role:** Set – prompt construction for header aggregation rules.

#### Node: **AI Analyze All Headers**
- **Type/role:** LangChain LLM Chain node – asks LLM for aggregation rules.
- **Upstream model:** OpenAI LLM node connected via `ai_languageModel`.
- **Failure cases:** auth, rate limit, response not JSON, token overflow (if groups are large).

#### Node: **Process AI Response1**
- **Type/role:** Code – parses AI output into `aggregationRules`, merges with defaults:
  - volumetric fields → `sum`
  - price-like → `mean`
  - others → `last`
- **Important:** supports several possible AI response fields (`text`, `content`, etc.) and strips markdown fences.
- **Failure cases:** invalid JSON; falls back to defaults.

#### Node: **Group Data with AI Rules1**
- **Type/role:** Code – groups `rawData` by `groupByParam` (mapped using `headerMapping`).
- **Aggregation behavior:** per header rule: `sum`, `mean/average`, `last`, `first`.
- **Failure cases:** missing group-by value excludes rows; numeric parsing uses `parseFloat` with comma-to-dot replacement.

#### Node: **Find Category Fields**
- **Type/role:** Code – detects which field represents “category” (Category/IFC/etc.) and lists volumetric fields.
- **Outputs:** `categoryField`, `categoryValues`, `volumetricFields`, `groupedData`.

#### Node: **CONFIG - AI Classify**
- **Type/role:** Set – prompt to classify categories building/non-building.

#### Node: **AI Classify Categories**
- **Type/role:** LangChain LLM Chain – returns JSON classification list (expected).
- **Failure cases:** response format mismatch.

#### Node: **Apply Classification to Groups**
- **Type/role:** Code – applies classification to each group and sets:
  - `is_building_element`, `element_confidence`, `element_reason`
- **Hard exclude:** levels/grids/annotations/views/sheets/RPC/entourage/groups etc. always excluded.
- **Failure cases:**
  - AI response parsing failure → fallback heuristics by keyword or volumetric presence.
  - Category field absent → relies on volumetric detection.

#### Node: **Is Building Element1**
- **Type/role:** IF – passes only `is_building_element === true`.
- **Outputs:**
  - True → STAGE 0 - Collect BIM Data
  - False → Non-Building Elements Output1

#### Node: **Non-Building Elements Output1**
- **Type/role:** Set – informational output that elements were excluded.

---

### Block 3 — Project Analysis (Stages 0–3)

**Overview:** Converts grouped BIM data into a deduplicated list of element types, then uses AI to infer project type and a construction phase plan, and assigns types to phases.

**Nodes involved**
- STAGE 0 - Collect BIM Data
- CONFIG - STAGE 1
- STAGE 1 - Detect Project Type
- Parse Stage 1 - Project Type
- CONFIG - STAGE 2
- STAGE 2 - Generate Construction Phases
- Parse Stage 2 - Phases
- STAGE 2.5 - Compact Types1
- CONFIG - STAGE 5
- STAGE 3 - Assign Types to Phases1
- Parse Stage 3 - Final Structure
- Prepare Types for Decomposition

#### Node: **STAGE 0 - Collect BIM Data**
- **Type/role:** Code – deduplicates BIM “types” by `(type_name, category)` and aggregates quantities.
- **Parameter classification:** volumetric/material/identifier/system/other via regex.
- **Deduplication:** Map keyed by `type_name|||category`.
- **Outputs:** `raw_types` array with aggregated quantities, counts, and parameter info.
- **Failure cases:** missing `Type Name` or `Category` reduces quality; capped to `MAX_TYPES=1000`.

#### Node: **CONFIG - STAGE 1** / **STAGE 1 - Detect Project Type**
- **Type/role:** Set + LLM Chain – determines project type and scale from overall composition.
- **Failure cases:** response must be JSON string; parsing node handles fallback.

#### Node: **Parse Stage 1 - Project Type**
- **Type/role:** Code – parses `.text` into JSON; falls back to safe defaults.
- **Outputs:** project_type object, scale, categories, notes plus pass-through Stage 0 data.

#### Node: **CONFIG - STAGE 2** / **STAGE 2 - Generate Construction Phases**
- **Type/role:** Set + LLM Chain – creates a phased plan with localized phase names.
- **Critical requirement:** phase_name must be in the selected language.
- **Failure cases:** token size (phase selection prompt is large), formatting mismatch.

#### Node: **Parse Stage 2 - Phases**
- **Type/role:** Code – parses `.text` JSON; outputs `construction_phases`, `phase_logic`.

#### Node: **STAGE 2.5 - Compact Types1**
- **Type/role:** Code – reduces token load by limiting quantities to key fields.
- **Outputs:** `compact_types` for LLM Stage 3 plus original `raw_types` for later.

#### Node: **CONFIG - STAGE 5** / **STAGE 3 - Assign Types to Phases1**
- **Type/role:** Set + LLM Chain – assigns each type to a phase id and sequence.

#### Node: **Parse Stage 3 - Final Structure**
- **Type/role:** Code – **forces completeness**: uses *all* BIM types from Stage 2 data and overlays LLM assignments where available.
- **Outputs:**
  - `types_with_phases` (complete list)
  - `phases_with_types` enriched with assigned types
- **Failure cases:** LLM returns partial list; this node compensates.

#### Node: **Prepare Types for Decomposition**
- **Type/role:** Code – creates one item per type for looping through Stage 4.
- **Key behavior:** merges `quantities` + `additional_parameters` into a single `quantities` object for the LLM.
- **Outputs:** items containing type + phase + language/qdrant config.

---

### Block 4 — Work Decomposition Loop (Stage 4)

**Overview:** Iterates each type, stores it in static cache (race-condition safe), asks LLM to produce work items, then parses and normalizes work items (dedup, domain checks, unit fixes, fallbacks).

**Nodes involved**
- Loop Types for Decomposition
- Save Type Before LLM
- CONFIG - STAGE 4
- STAGE 4 - Decompose Type to Works
- Parse Decomposition & Prepare Works
- Has Work Items?
- Handle No Works

#### Node: **Loop Types for Decomposition**
- **Type/role:** Split In Batches – iterates types.
- **Config:** `reset: false` (keeps state across runs in a single execution).
- **Downstream:** feeds Save Type Before LLM and also connects to STAGE 8 (final aggregation) after the loop completes.
- **Failure cases:** if not looped properly, STAGE 8 may run prematurely depending on n8n execution order.

#### Node: **Save Type Before LLM**
- **Type/role:** Code – stores the current type into `workflow staticData.global.type_cache[typeKey]` and sets `current_type_key`.
- **Purpose:** prevents data mixing when multiple items are processed.
- **Failure cases:** static data persists across workflow executions unless cleared later.

#### Node: **CONFIG - STAGE 4** / **STAGE 4 - Decompose Type to Works**
- **Type/role:** Set + LLM Chain – generates a category-respecting work plan for each type.
- **Constraints enforced in prompt:** min 2 works; language requirement; quantity rules.
- **Failure cases:** hallucinated domains; parsing node handles via validation/fallbacks.

#### Node: **Parse Decomposition & Prepare Works**
- **Type/role:** Code – parses LLM response for each type and expands to work items.
- **Key safety mechanisms:**
  - Domain mismatch filtering (electrical/industrial/vertical transport)
  - Fallback generation if no works survive
  - Unit-based category fix: windows/doors/etc. forced to `element_count` instead of area/volume
  - Work dedup by name/query and re-sequencing
- **Outputs:** one item per work with `_typeKey` and type context.

#### Node: **Has Work Items?**
- **Type/role:** IF – checks `no_works !== true`.
- **Outputs:**
  - True → Loop Work Items
  - False → Handle No Works

#### Node: **Handle No Works**
- **Type/role:** Code – creates a “no works” type result (zero cost) for downstream aggregation.

---

### Block 5 — Pricing & Calculation (Stages 5–7)

**Overview:** For each work item, build multiple search strategies, embed text, query Qdrant, pick best rate, map units/quantities, and compute total costs/resources.

**Nodes involved**
- Loop Work Items
- STAGE 5.1 - Prepare Search Strategies1
- Rate Limit Wait
- STAGE 5.1 - Embeddings
- STAGE 5.1 - Vector Search
- STAGE 5.2 - Parse Results
- STAGE 6 - Map Rate Units to BIM
- STAGE 7 - Calculate Costs
- Accumulate Work Results

#### Node: **Loop Work Items**
- **Type/role:** Split In Batches – iterates work items per type.
- **Connections:** triggers both aggregation path and pricing path:
  - to Aggregate Type Works (type-level aggregation)
  - to STAGE 5.1 - Prepare Search Strategies1 (pricing pipeline)
- **Potential issue:** dual outgoing edges can cause ordering complexity; type aggregation relies on staticData accumulation.

#### Node: **STAGE 5.1 - Prepare Search Strategies1**
- **Type/role:** Code – stores full work data in static cache (`work_items_cache`) and emits a compact search payload:
  - strategies: exact, work+unit, key terms, category-focused
- **Purpose:** minimize payload size for vector search and LLM steps.
- **Failure cases:** cache key collisions if `_typeKey`/work identifiers are inconsistent.

#### Node: **Rate Limit Wait**
- **Type/role:** Wait – throttles before vector search (0.5 seconds).
- **Failure cases:** none; helps avoid API throttling.

#### Node: **STAGE 5.1 - Embeddings**
- **Type/role:** OpenAI Embeddings – `text-embedding-3-large`, 3072 dims.
- **Credentials:** OpenAI API credential required.
- **Failure cases:** API quota, invalid key, dimension mismatch if Qdrant collection uses different size.

#### Node: **STAGE 5.1 - Vector Search**
- **Type/role:** Qdrant Vector Store (load/search).
- **Key config:**
  - `topK: 5`
  - `qdrantCollection`: dynamic from `qdrant_collection`
  - `contentPayloadKey: content`
- **Credentials:** Qdrant API credential required.
- **Failure cases:** collection missing, Qdrant unavailable, embedding size mismatch.

#### Node: **STAGE 5.2 - Parse Results**
- **Type/role:** Code – parses Qdrant result payloads into rate candidates and selects best by quality scoring:
  - has price, has resources, unit match, material match, work-type keyword match, “complete rate” bonus.
- **Output:** `rateInfo`, categorized `resources`, `all_search_results` (debug).
- **Failure cases:** unexpected content format, missing metadata fields, zero results → returns NOT_FOUND.

#### Node: **STAGE 6 - Map Rate Units to BIM**
- **Type/role:** Code – maps BIM quantities to rate units; supports:
  - direct, derived, and formula methods
  - unit divisors (10/100/1000)
  - unit conversion heuristics (mm→m, mm²→m², mm³→m³)
- **Critical bug risk:** the mm³ conversion factor is set to `1+1234567890` (not 1,000,000,000). This will severely distort volume conversions. It should typically be **1,000,000,000** for mm³→m³.
- **Failure cases:** `eval()` used for formulas (sanitized but still risky); missing parameters falls back to element_count.

#### Node: **STAGE 7 - Calculate Costs**
- **Type/role:** Code – computes:
  - total cost = quantity_in_rate_units × unit_cost
  - breaks down into worker/machine/material/machinist/electricity
  - scales each resource by quantity
  - emits quality indicators and formatted values.
- **Failure cases:** missing breakdown; uses heuristic 60/40 split if needed.

#### Node: **Accumulate Work Results**
- **Type/role:** Code – appends work result into `staticData.work_results[typeKey]`.
- **Failure cases:** static data not cleaned can cross-contaminate runs; later Stage 8 clears.

---

### Block 6 — Validation & Aggregation (Stages 7.5–8)

**Overview:** Aggregates all work results per type, optionally validates completeness with an LLM, stores the type result, and finally aggregates all types into construction phases (with category-based correction).

**Nodes involved**
- Aggregate Type Works
- CONFIG - STAGE 7.5
- STAGE 7.5 - Validate Type Works
- STAGE 7.5 - Parse Validation
- Store Type Result
- STAGE 8 - Aggregate by Phases

#### Node: **Aggregate Type Works**
- **Type/role:** Code – reads `staticData.current_type_key` and collects `staticData.work_results[typeKey]`.
- **Outputs:** type summary: cost totals, labor totals, quality distribution, avg confidence.
- **Side effects:** deletes work_results and type_cache entries for the type to avoid growth.
- **Failure cases:** if `current_type_key` not set properly, may aggregate wrong type.

#### Node: **CONFIG - STAGE 7.5** / **STAGE 7.5 - Validate Type Works**
- **Type/role:** Set + LLM Chain – asks an LLM to validate completeness, duplication, sequencing, missing works.
- **Failure cases:** can be expensive; output parser is tolerant.

#### Node: **STAGE 7.5 - Parse Validation**
- **Type/role:** Code – parses validation JSON and sets a final status based on completeness score.
- **Output:** attaches `validation` and `quality_summary` to aggregated type.

#### Node: **Store Type Result**
- **Type/role:** Code – pushes type result into `staticData.accumulated_types`.
- **Also loops:** output goes back to Loop Types for Decomposition to continue iteration.

#### Node: **STAGE 8 - Aggregate by Phases**
- **Type/role:** Code – final report-level aggregation:
  - ensures no types are missing (fallback using Stage 3 list)
  - calculates grand totals
  - assigns/corrects phases by a hard-coded category→phase mapping (overrides LLM mistakes)
  - builds `by_phase`, `by_type`, `types_without_works`, `quality_report`
  - clears `staticData.accumulated_types` and `staticData.work_results`.
- **Failure cases:** category strings differ (localization/export), causing more items to fall into UNASSIGNED.

---

### Block 7 — Report Generation (Stage 9)

**Overview:** Generates a multi-language professional HTML report and an Excel-compatible XLS (HTML content), saves both in the same folder as the Revit project, and opens the HTML file.

**Nodes involved**
- STAGE 9 - Generate Cost Estimate
- Save to Project Folder
- Write XLS File
- Write HTML File
- Open HTML in Browser

#### Node: **STAGE 9 - Generate Cost Estimate**
- **Type/role:** Code – builds HTML report with:
  - KPIs, charts, collapsible hierarchy, rate links
  - multi-language translation dictionary
  - resource tagging (labor/material/machine)
  - embeds both HTML and XLS binaries
- **Output:** `binary.html` and `binary.xls` plus filenames and metadata.
- **Failure cases:** very large projects → HTML can be huge; memory/binary limits.

#### Node: **Save to Project Folder**
- **Type/role:** Code – builds `html_path` and `xls_path` based on folder of `project_file`.
- **Fallback:** uses `./output.rvt` if not found.
- **Failure cases:** missing permissions; invalid characters in filenames.

#### Node: **Write HTML File**
- **Type/role:** Write Binary File – writes `binary.html` to `html_path`.

#### Node: **Write XLS File**
- **Type/role:** Write Binary File – writes `binary.xls` to `xls_path`.

#### Node: **Open HTML in Browser**
- **Type/role:** Execute Command – Windows `start "" "<html_path>"`.
- **n8n 2.0+ requirement:** Execute Command must be enabled via `NODES_EXCLUDE=[]`.
- **Failure cases:** non-Windows environment; path quoting.

---

### Supporting / Optional / Disabled Nodes

These nodes exist and must be considered if you enable them:

- **Limit to 10 Groups** (disabled): would slice grouped types to first 10 for faster testing.
- **Export All Outputs** + **Write Output File** (both disabled): dump outputs of many nodes into a text/binary file for debugging.
- **Additional LLM model nodes** (disabled): OpenRouter, xAI Grok, Anthropic, Gemini, DeepSeek; can replace OpenAI LLM by connecting them to LLM Chain nodes.
- **Sticky note nodes**: documentation overlays only (no execution).

---

## 3. Summary Table (All Nodes)

> Notes:  
> - “Sticky Note” column reproduces visible sticky note content relevant to the node’s area.  
> - Sticky notes that span large blocks are repeated for nodes within those blocks conceptually (conversion/data loading/stages/pricing/validation/reporting).  
> - Disabled nodes are included.

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | manualTrigger | Manual entry point | — | Setup - Define file paths1 | ## ⚡️ Execute workflow |
| Setup - Define file paths1 | set | User config: converter path, RVT file, grouping, language | When clicking 'Execute workflow' | Configure Language & Vector DB | ## ⚙️ Configuration (paths, language_code list, pricing levels) |
| Configure Language & Vector DB | code | Map language→currency/locale/Qdrant collection/system prompts | Setup - Define file paths1 | Create - Excel filename1 | ## ⚙️ Configuration |
| Create - Excel filename1 | set | Derive xlsx output filename and pass config | Configure Language & Vector DB | Check - Does Excel file exist?1 | ## Block 1: Conversion Block |
| Check - Does Excel file exist?1 | readBinaryFile | Detect whether export exists | Create - Excel filename1 | If - File exists?1 | ## Block 1: Conversion Block |
| If - File exists?1 | if | Branch: skip conversion vs run converter | Check - Does Excel file exist?1 | Info - Skip conversion1; Extract - Run converter1 | ## Block 1: Conversion Block |
| Info - Skip conversion1 | set | Emit status: already exists | If - File exists?1 (true) | Merge - Continue workflow1 | ## Block 1: Conversion Block |
| Extract - Run converter1 | executeCommand | Run RvtExporter.exe | If - File exists?1 (false) | Check - Did extraction succeed?1 | ## Block 1: Conversion Block / ## ⚠️ n8n 2.0+ Setup Required ⚠️ (Execute Command enabled via `NODES_EXCLUDE=[]`) |
| Check - Did extraction succeed?1 | if | Route success vs error from converter | Extract - Run converter1 | Error - Show what went wrong1; Set xlsx_filename after success1 | ## Block 1: Conversion Block |
| Error - Show what went wrong1 | set | Provide error details | Check - Did extraction succeed?1 | Merge - Continue workflow1 | ## Block 1: Conversion Block |
| Set xlsx_filename after success1 | set | Propagate filename after conversion | Check - Did extraction succeed?1 | Merge - Continue workflow1 | ## Block 1: Conversion Block |
| Merge - Continue workflow1 | merge | Rejoin skip/success/error | Info - Skip conversion1; Error - Show what went wrong1; Set xlsx_filename after success1 | Set Parameters1 | ## Block 1: Conversion Block |
| Set Parameters1 | set | Standardize `path_to_file` | Merge - Continue workflow1 | Read Excel File1 | ## Block 2: Data Loading & AI Classification |
| Read Excel File1 | readBinaryFile | Load XLSX binary | Set Parameters1 | Parse Excel1 | ## Block 2: Data Loading & AI Classification |
| Parse Excel1 | spreadsheetFile | Parse XLSX → JSON rows | Read Excel File1 | On the standard 3D View | ## Block 2: Data Loading & AI Classification |
| On the standard 3D View | if | Filter rows visible in standard 3D view | Parse Excel1 | Extract Headers and Data; Non-3D View Elements Output1 | ## Block 2: Data Loading & AI Classification |
| Non-3D View Elements Output1 | set | Output filtered-out count | On the standard 3D View (false) | — | Elements not visible in 3D view |
| Extract Headers and Data | code | Collect/clean headers, mapping, rawData | On the standard 3D View (true) | CONFIG - AI Headers | ### 🤖 AI Classification of parameters for selecting the aggregation method |
| CONFIG - AI Headers | set | Build LLM prompt for aggregation rules | Extract Headers and Data | AI Analyze All Headers | ### 🤖 AI Classification of parameters for selecting the aggregation method |
| AI Analyze All Headers | chainLlm | LLM: header aggregation rules | CONFIG - AI Headers | Process AI Response1 | ### 🤖 AI Classification of parameters for selecting the aggregation method |
| Process AI Response1 | code | Parse AI rules + defaults | AI Analyze All Headers | Group Data with AI Rules1 | ## Block 2: Data Loading & AI Classification |
| Group Data with AI Rules1 | code | Group rows by Type Name and aggregate | Process AI Response1 | Find Category Fields | ## Block 2: Data Loading & AI Classification / ## Only 10 Groups (testing limiter) |
| Limit to 10 Groups | code (disabled) | Debug: limit grouped output | (would be after grouping) | (downstream) | ## Only 10 Groups (Limits output to first 10 element groups…) |
| Find Category Fields | code | Detect category field + volumetric fields | Group Data with AI Rules1 | CONFIG - AI Classify | ### 🤖  AI classification (optional) highlighting of building elements |
| CONFIG - AI Classify | set | Prompt for building vs non-building category classification | Find Category Fields | AI Classify Categories | ### 🤖  AI classification (optional) highlighting of building elements |
| AI Classify Categories | chainLlm | LLM classify categories | CONFIG - AI Classify | Apply Classification to Groups | ### 🤖  AI classification (optional) highlighting of building elements |
| Apply Classification to Groups | code | Apply AI classification + hard excludes | AI Classify Categories | Is Building Element1 | ## Block 2: Data Loading & AI Classification |
| Is Building Element1 | if | Filter building elements only | Apply Classification to Groups | STAGE 0 - Collect BIM Data; Non-Building Elements Output1 | ## Block 2: Data Loading & AI Classification |
| Non-Building Elements Output1 | set | Inform excluded non-building elements | Is Building Element1 (false) | — | Non-building elements excluded from cost estimation |
| STAGE 0 - Collect BIM Data | code | Deduplicate/aggregate BIM types | Is Building Element1 (true) | CONFIG - STAGE 1 | ## Block 3: Project Analysis (Stages 0-3) |
| CONFIG - STAGE 1 | set | Prompt: detect project type | STAGE 0 - Collect BIM Data | STAGE 1 - Detect Project Type | ### 🤖 AI Project Type Detection (renovation, construction, demolition) |
| STAGE 1 - Detect Project Type | chainLlm | LLM project type & scale | CONFIG - STAGE 1 | Parse Stage 1 - Project Type | ### 🤖 AI Project Type Detection (renovation, construction, demolition) |
| Parse Stage 1 - Project Type | code | Parse Stage 1 JSON | STAGE 1 - Detect Project Type | CONFIG - STAGE 2 | ## Block 3: Project Analysis (Stages 0-3) |
| CONFIG - STAGE 2 | set | Prompt: generate phases | Parse Stage 1 - Project Type | STAGE 2 - Generate Construction Phases | ### 🤖 AI Developing a step-by-step work plan |
| STAGE 2 - Generate Construction Phases | chainLlm | LLM phase plan | CONFIG - STAGE 2 | Parse Stage 2 - Phases | ### 🤖 AI Developing a step-by-step work plan |
| Parse Stage 2 - Phases | code | Parse phases JSON | STAGE 2 - Generate Construction Phases | STAGE 2.5 - Compact Types1 | ## Block 3: Project Analysis (Stages 0-3) |
| STAGE 2.5 - Compact Types1 | code | Reduce types payload for Stage 3 | Parse Stage 2 - Phases | CONFIG - STAGE 5 | ### 🤖 AI Mapping Element Types → Work Plan Stages |
| CONFIG - STAGE 5 | set | Prompt: assign types to phases | STAGE 2.5 - Compact Types1 | STAGE 3 - Assign Types to Phases1 | ### 🤖 AI Mapping Element Types → Work Plan Stages |
| STAGE 3 - Assign Types to Phases1 | chainLlm | LLM assigns types to phases | CONFIG - STAGE 5 | Parse Stage 3 - Final Structure | ### 🤖 AI Mapping Element Types → Work Plan Stages |
| Parse Stage 3 - Final Structure | code | Force full type list + enrich phases | STAGE 3 - Assign Types to Phases1 | Prepare Types for Decomposition | ## Block 3: Project Analysis (Stages 0-3) |
| Prepare Types for Decomposition | code | Create per-type items for Stage 4 loop | Parse Stage 3 - Final Structure | Loop Types for Decomposition | ## Block 4: Work Decomposition Loop |
| Loop Types for Decomposition | splitInBatches | Iterate types | Prepare Types for Decomposition | Save Type Before LLM; STAGE 8 - Aggregate by Phases | ## Block 4: Work Decomposition Loop |
| Save Type Before LLM | code | Static cache type data safely | Loop Types for Decomposition | CONFIG - STAGE 4 | ## Block 4: Work Decomposition Loop |
| CONFIG - STAGE 4 | set | Prompt: decompose type → works | Save Type Before LLM | STAGE 4 - Decompose Type to Works | ### 🤖 AI work-scope generation from building element types |
| STAGE 4 - Decompose Type to Works | chainLlm | LLM work decomposition | CONFIG - STAGE 4 | Parse Decomposition & Prepare Works | ### 🤖 AI work-scope generation from building element types |
| Parse Decomposition & Prepare Works | code | Parse/dedup/fallback works | STAGE 4 - Decompose Type to Works | Has Work Items? | ## Block 4: Work Decomposition Loop |
| Has Work Items? | if | Route works vs no-works | Parse Decomposition & Prepare Works | Loop Work Items; Handle No Works | ## Block 4: Work Decomposition Loop |
| Handle No Works | code | Create zero-cost type output | Has Work Items? (false) | Store Type Result | ## Block 6: Validation & Aggregation |
| Loop Work Items | splitInBatches | Iterate work items | Has Work Items? (true) | Aggregate Type Works; STAGE 5.1 - Prepare Search Strategies1 | ## Block 5: Pricing & Calculation (Stages 5-7) |
| STAGE 5.1 - Prepare Search Strategies1 | code | Build multi-query strategies + cache work | Loop Work Items | Rate Limit Wait | ## Block 5: Pricing & Calculation (Stages 5-7) |
| Rate Limit Wait | wait | Throttle vector search | STAGE 5.1 - Prepare Search Strategies1 | STAGE 5.1 - Vector Search | ## Block 5: Pricing & Calculation (Stages 5-7) |
| STAGE 5.1 - Embeddings | embeddingsOpenAi | Produce embeddings (3072 dims) | (AI embedding connection) | STAGE 5.1 - Vector Search | ### 🤖 AI translate Text-qery in emmbeding |
| STAGE 5.1 - Vector Search | vectorStoreQdrant | Qdrant semantic search | Rate Limit Wait | STAGE 5.2 - Parse Results | ### 📥 To enable vector database search… (install Qdrant, set creds, load datasets…) |
| STAGE 5.2 - Parse Results | code | Parse candidates + score best rate | STAGE 5.1 - Vector Search | STAGE 6 - Map Rate Units to BIM | ## Block 5: Pricing & Calculation (Stages 5-7) |
| STAGE 6 - Map Rate Units to BIM | code | Quantity/unit mapping + conversion | STAGE 5.2 - Parse Results | STAGE 7 - Calculate Costs | ## Block 5: Pricing & Calculation (Stages 5-7) |
| STAGE 7 - Calculate Costs | code | Cost + resources scaling | STAGE 6 - Map Rate Units to BIM | Accumulate Work Results | ## Block 5: Pricing & Calculation (Stages 5-7) |
| Accumulate Work Results | code | Append work result into static store | STAGE 7 - Calculate Costs | Loop Work Items | ## Block 5: Pricing & Calculation (Stages 5-7) |
| Aggregate Type Works | code | Aggregate all works for current type | Loop Work Items | CONFIG - STAGE 7.5 | ## Block 6: Validation & Aggregation |
| CONFIG - STAGE 7.5 | set | Prompt: validate estimate completeness | Aggregate Type Works | STAGE 7.5 - Validate Type Works | ### 🤖 AI verification of the quality of received data |
| STAGE 7.5 - Validate Type Works | chainLlm | LLM validation | CONFIG - STAGE 7.5 | STAGE 7.5 - Parse Validation | ### 🤖 AI verification of the quality of received data |
| STAGE 7.5 - Parse Validation | code | Parse validation JSON and score | STAGE 7.5 - Validate Type Works | Store Type Result | ## Block 6: Validation & Aggregation |
| Store Type Result | code | Store type result for Stage 8 | STAGE 7.5 - Parse Validation; Handle No Works | Loop Types for Decomposition | ## Block 6: Validation & Aggregation |
| STAGE 8 - Aggregate by Phases | code | Final phase aggregation + cleanup | Loop Types for Decomposition (loop completion) | STAGE 9 - Generate Cost Estimate | ## Block 6: Validation & Aggregation |
| STAGE 9 - Generate Cost Estimate | code | Build HTML+XLS report binaries | STAGE 8 - Aggregate by Phases | Save to Project Folder | ## Block 7: Report Generation |
| Save to Project Folder | code | Determine output folder + file paths | STAGE 9 - Generate Cost Estimate | Write HTML File; Write XLS File | ## Block 7: Report Generation |
| Write HTML File | writeBinaryFile | Write HTML to disk | Save to Project Folder | Open HTML in Browser | ## 📁 Output Files (Saved to project folder …) |
| Write XLS File | writeBinaryFile | Write XLS to disk | Save to Project Folder | — | ## 📁 Output Files (Saved to project folder …) |
| Open HTML in Browser | executeCommand | Open HTML via OS command | Write HTML File | Export All Outputs | ## Block 7: Report Generation / ## ⚠️ n8n 2.0+ Setup Required ⚠️ |
| Export All Outputs | code (disabled) | Debug: dump many node outputs | Open HTML in Browser | Write Output File | ## 📁 Debug node`s output |
| Write Output File | writeBinaryFile (disabled) | Save exported debug dump | Export All Outputs | — | ## 📁 Debug node`s output |
| OpenAI LLM | lmChatOpenAi | Primary LLM model for all chain nodes | — (AI connections) | Many chainLlm nodes | ## 🧠 Available AI Models |
| OpenRouter Chat Model1 | lmChatOpenRouter (disabled) | Alternate LLM provider | — | — | ## 🧠 Available AI Models |
| xAI Grok Chat Model1 | lmChatXAiGrok (disabled) | Alternate LLM provider | — | — | ## 🧠 Available AI Models |
| Anthropic Chat Model2 | lmChatAnthropic (disabled) | Alternate LLM provider | — | — | ## 🧠 Available AI Models |
| Google Gemini Chat Model | lmChatGoogleGemini (disabled) | Alternate LLM provider | — | — | ## 🧠 Available AI Models |
| DeepSeek Chat Model | lmChatDeepSeek (disabled) | Alternate LLM provider | — | — | ## 🧠 Available AI Models |
| Header | stickyNote | Visual header note | — | — | ## 🚀 CAD (BIM) Cost Estimation Pipeline… GitHub link |
| Sticky Note1 | stickyNote | GitHub star request | — | — | ⭐ Star repository: https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR |
| Configuration | stickyNote | Setup instructions (language & DB) | — | — | (contains language_code list and config hints) |
| Pipeline Overview | stickyNote | Stage table overview | — | — | Stage 0–9 overview table |
| Block 1 - Conversion | stickyNote | Explains conversion block | — | — | Conversion block explanation |
| Block 2 - Data Loading | stickyNote | Explains loading/classification block | — | — | Data loading & AI classification explanation |
| Block 2 - Data Loading1 | stickyNote | Empty overlay | — | — |  |
| Sticky Note3 | stickyNote | AI header aggregation note | — | — | ### 🤖 AI Classification of parameters for selecting the aggregation method |
| Sticky Note | stickyNote | AI building element highlighting | — | — | ### 🤖  AI classification (optional) highlighting of building elements |
| Block 3 - Stages 0-3 | stickyNote | Explains project analysis | — | — | Project analysis stages explanation |
| Block 3 - Stages 0- | stickyNote | Empty overlay | — | — |  |
| Sticky Note2 | stickyNote | AI project type detection | — | — | ### 🤖 AI Project Type Detection (renovation, construction, demolition) |
| Sticky Note4 | stickyNote | AI phase planning | — | — | ### 🤖 AI Developing a step-by-step work plan |
| Sticky Note6 | stickyNote | AI mapping types→phases | — | — | ### 🤖 AI Mapping Element Types → Work Plan Stages |
| Block 4 - Decomposition1 | stickyNote | Work decomposition block description | — | — | Work decomposition loop explanation |
| Block 4 - Decomposition | stickyNote | Empty overlay | — | — |  |
| Sticky Note7 | stickyNote | Work-scope generation note | — | — | ### 🤖 AI work-scope generation from building element types |
| Block 5 - Pricing | stickyNote | Pricing & calculation explanation | — | — | Pricing stages explanation |
| Sticky Note10 | stickyNote | Qdrant enablement instructions | — | — | Qdrant setup requirements (install, creds, datasets, collections) |
| Sticky Note11 | stickyNote | Embedding note | — | — | ### 🤖 AI translate Text-qery in emmbeding |
| Sticky Note8 | stickyNote | Search note | — | — | ### 🔍 Search for jobs in a vector database |
| Block 6 - Validation | stickyNote | Validation & aggregation explanation | — | — | Validation and aggregation explanation |
| Sticky Note9 | stickyNote | AI verification note | — | — | ### 🤖 AI verification of the quality of received data |
| Block 7 - Reports | stickyNote | Reporting explanation | — | — | Report generation explanation |
| Output Files | stickyNote | Example output filenames | — | — | Output HTML/XLS saved to project folder |
| Output Files1 | stickyNote | Debug outputs note | — | — | ## 📁 Debug node`s output |
| LLM Models | stickyNote | List of supported models | — | — | Available AI Models list |
| Sticky Note5 | stickyNote | Only 10 groups note | — | — | Limits output to first 10 element groups… |
| Sticky Note13 | stickyNote | n8n 2.0 Execute Command enablement | — | — | https://docs.n8n.io/2-0-breaking-changes/ |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

1) **Create workflow**
   - New workflow name: *Estimate 4D/5D construction costs from Revit BIM models with DDC CWICR*.

2) **Add entry**
   - Add **Manual Trigger** → name: `When clicking 'Execute workflow'`.

3) **Add configuration (paths + language)**
   - Add **Set** node → `Setup - Define file paths1`
     - `path_to_converter` (string): full path to `RvtExporter.exe`
     - `project_file` (string): full path to `.rvt`
     - `group_by` (string): `Type Name`
     - `user_project_description` (string): optional
     - `language_code` (string): one of `DE/EN/FR/ES/HI/PT/RU/ZH/AR`

4) **Add language/vector DB config**
   - Add **Code** node → `Configure Language & Vector DB`
   - Implement language mapping logic:
     - output: `qdrant_collection`, `currency`, `locale`, `system_prompt_lang`, `search_lang`, `pricing_level`, etc.

5) **Derive Excel filename**
   - Add **Set** node → `Create - Excel filename1`
   - Create `xlsx_filename` from `project_file` (remove extension and append `_<ext>.xlsx`).

6) **Excel existence check**
   - Add **Read Binary File** → `Check - Does Excel file exist?1`
     - `filePath`: `={{ $json["xlsx_filename"] }}`
     - enable **Continue On Fail** and **Always Output Data**.
   - Add **IF** → `If - File exists?1`
     - condition: `={{ $binary.data ? true : false }}` equals `true`.

7) **Skip path**
   - Add **Set** → `Info - Skip conversion1` (status + pass filename).

8) **Conversion path (critical n8n 2.0+)**
   - Add **Execute Command** → `Extract - Run converter1`
     - command: `"{{path_to_converter}}" "{{project_file}}"`
   - **IMPORTANT:** enable Execute Command in n8n 2.0+ by setting `NODES_EXCLUDE=[]` (env) and restarting n8n.

9) **Extraction success routing**
   - Add **IF** → `Check - Did extraction succeed?1` (test converter error presence).
   - Add **Set** → `Error - Show what went wrong1`.
   - Add **Set** → `Set xlsx_filename after success1`.

10) **Merge conversion and skip**
   - Add **Merge** → `Merge - Continue workflow1` and connect:
     - Skip branch output
     - Conversion success output
     - Conversion error output (if you want workflow to proceed; otherwise stop)

11) **Set standard parameter**
   - Add **Set** → `Set Parameters1`
     - `path_to_file = {{$json.xlsx_filename}}`

12) **Load & parse Excel**
   - Add **Read Binary File** → `Read Excel File1` using `path_to_file`.
   - Add **Spreadsheet File** → `Parse Excel1`
     - `fileFormat: xlsx`
     - `headerRow: true`
     - Ensure `sheetName` is set (either hardcode or add a Set node that defines `sheet_name`).

13) **Filter 3D view**
   - Add **IF** → `On the standard 3D View`
     - condition: `On the standard 3D View : Boolean` is true.
   - Add **Set** → `Non-3D View Elements Output1` (optional reporting).

14) **Extract headers and raw data**
   - Add **Code** → `Extract Headers and Data` (collect headers, mapping, rawData).

15) **AI aggregation rules**
   - Add **Set** → `CONFIG - AI Headers` to build the prompt.
   - Add **LLM Chain** → `AI Analyze All Headers`.
   - Add **Chat Model** (OpenAI recommended) and connect it as the **AI language model**:
     - Add `OpenAI LLM` node with model `chatgpt-4o-latest`, temperature 0.
     - Configure **OpenAI credentials**.
   - Add **Code** → `Process AI Response1` to parse rules and apply defaults.

16) **Group data**
   - Add **Code** → `Group Data with AI Rules1` to aggregate by `group_by`.
   - (Optional) Add a **Code** node `Limit to 10 Groups` and slice results for debugging.

17) **Find category field**
   - Add **Code** → `Find Category Fields`.

18) **AI classify categories**
   - Add **Set** → `CONFIG - AI Classify`.
   - Add **LLM Chain** → `AI Classify Categories`.
   - Add **Code** → `Apply Classification to Groups` including hard exclude list.

19) **Filter building elements**
   - Add **IF** → `Is Building Element1` (`is_building_element === true`).
   - Add **Set** → `Non-Building Elements Output1` (optional).

20) **Stages 0–3**
   - Add **Code** → `STAGE 0 - Collect BIM Data`.
   - Add **Set** + **LLM Chain** → `CONFIG - STAGE 1` + `STAGE 1 - Detect Project Type`.
   - Add **Code** → `Parse Stage 1 - Project Type`.
   - Add **Set** + **LLM Chain** → `CONFIG - STAGE 2` + `STAGE 2 - Generate Construction Phases`.
   - Add **Code** → `Parse Stage 2 - Phases`.
   - Add **Code** → `STAGE 2.5 - Compact Types1`.
   - Add **Set** + **LLM Chain** → `CONFIG - STAGE 5` + `STAGE 3 - Assign Types to Phases1`.
   - Add **Code** → `Parse Stage 3 - Final Structure`.
   - Add **Code** → `Prepare Types for Decomposition`.

21) **Type loop + decomposition**
   - Add **Split In Batches** → `Loop Types for Decomposition`.
   - Add **Code** → `Save Type Before LLM` (static cache).
   - Add **Set** + **LLM Chain** → `CONFIG - STAGE 4` + `STAGE 4 - Decompose Type to Works`.
   - Add **Code** → `Parse Decomposition & Prepare Works`.
   - Add **IF** → `Has Work Items?`.
   - Add **Code** → `Handle No Works`.

22) **Work loop + pricing**
   - Add **Split In Batches** → `Loop Work Items`.
   - Add **Code** → `STAGE 5.1 - Prepare Search Strategies1`.
   - Add **Wait** → `Rate Limit Wait` (0.5s).
   - Add **OpenAI Embeddings** → `STAGE 5.1 - Embeddings`
     - model: `text-embedding-3-large`
     - dimensions: 3072
   - Add **Qdrant Vector Store** → `STAGE 5.1 - Vector Search`
     - mode: load/search
     - `topK: 5`
     - collection name driven by `qdrant_collection`
     - configure **Qdrant credentials**
   - Add **Code** → `STAGE 5.2 - Parse Results`.
   - Add **Code** → `STAGE 6 - Map Rate Units to BIM`  
     - Fix the mm³→m³ conversion factor to 1,000,000,000 to avoid catastrophic volume errors.
   - Add **Code** → `STAGE 7 - Calculate Costs`.
   - Add **Code** → `Accumulate Work Results`.

23) **Type aggregation + validation**
   - Add **Code** → `Aggregate Type Works`.
   - Add **Set** + **LLM Chain** → `CONFIG - STAGE 7.5` + `STAGE 7.5 - Validate Type Works`.
   - Add **Code** → `STAGE 7.5 - Parse Validation`.
   - Add **Code** → `Store Type Result` and connect it back to `Loop Types for Decomposition` to continue.

24) **Final aggregation**
   - Add **Code** → `STAGE 8 - Aggregate by Phases` connected from the type loop completion.

25) **Report generation**
   - Add **Code** → `STAGE 9 - Generate Cost Estimate`.
   - Add **Code** → `Save to Project Folder`.
   - Add **Write Binary File** → `Write HTML File` (data property `html`).
   - Add **Write Binary File** → `Write XLS File` (data property `xls`).
   - Add **Execute Command** → `Open HTML in Browser` (Windows `start`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| GitHub repository (DDC CWICR / OpenConstructionEstimate) | https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR |
| Execute Command disabled by default in n8n 2.0+; enable via `NODES_EXCLUDE=[]` | https://docs.n8n.io/2-0-breaking-changes/ |
| Qdrant is required for vector search; you must run an instance and load the datasets/collections | (see workflow sticky note instructions) |
| Output files saved next to the Revit project file | HTML + XLS (Excel-compatible) |
| Known risk: unit conversion bug in Stage 6 for mm³→m³ conversion factor | Review and correct before production use |

Disclaimer (as provided): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.*