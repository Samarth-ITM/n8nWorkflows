Generate and enrich Google Maps B2B leads with SDR-ready data in Google Sheets

https://n8nworkflows.xyz/workflows/generate-and-enrich-google-maps-b2b-leads-with-sdr-ready-data-in-google-sheets-14396


# Generate and enrich Google Maps B2B leads with SDR-ready data in Google Sheets

# 1. Workflow Overview

This workflow builds a lightweight B2B lead generation pipeline for SDR teams using Google Maps data. A user submits a business category and city through an n8n form, the workflow searches Google Places, enriches each result with phone/website details, cleans and consolidates the data, filters for higher-quality leads, and appends the final records into Google Sheets.

It is designed for use cases such as:

- Prospecting local businesses by vertical and geography
- Building outbound lead lists for SDR or sales operations teams
- Enriching Google Maps search results with contact-ready fields
- Logging validated leads into a simple operational datastore

## 1.1 Input Reception & Search Preparation

The workflow starts from an n8n form where the operator provides the business type and city. A Set node then normalizes those inputs into internal field names and stores the Google API key used later by the HTTP Request nodes.

## 1.2 Google Maps Search & Pagination

The workflow sends a Google Places Text Search request to the Google Maps Places API using the submitted business type and city. It uses pagination to continue requesting additional result pages as long as a `next_page_token` is returned.

## 1.3 Per-Place Detail Enrichment

The search response contains arrays of places. A Split Out node explodes the `results` array into individual items so each place can be processed separately. For each result, a second Google Places API call fetches selected details such as phone number and website.

## 1.4 Merge, Aggregation & Data Cleaning

The original search results and per-place details are merged and then aggregated so a Code node can process the whole dataset at once. The custom JavaScript normalizes names, builds a details map, merges the base and detail datasets, and outputs a consistent lead schema.

## 1.5 Lead Validation & Storage

An If node acts as a quality gate and only allows leads with an existing phone number to pass. Qualified leads are appended into a Google Sheets worksheet with explicit field-to-column mappings.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception & Search Preparation

### Overview
This block captures the user’s search request and prepares the fields needed by the rest of the workflow. It also stores the Google Places API key in the item payload so downstream HTTP requests can reference it.

### Nodes Involved
- Forms Trigger
- Format Search Query

### Node Details

#### Forms Trigger
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry point node that exposes an n8n-hosted form and starts the workflow when submitted.
- **Configuration choices:**  
  - Form title: `SDR`
  - Two fields are defined:
    - `Tipo de Negócio`
    - `Cidade`
- **Key expressions or variables used:**  
  None in the node itself; it emits user-submitted values.
- **Input and output connections:**  
  - No input
  - Output → `Format Search Query`
- **Version-specific requirements:**  
  Uses `typeVersion 2.3`; form behavior depends on newer n8n form capabilities.
- **Edge cases or potential failure types:**  
  - Empty or malformed form submissions
  - Users submitting overly broad categories or ambiguous city names
  - Form URL accessibility issues if n8n is not publicly reachable
- **Sub-workflow reference:**  
  None

#### Format Search Query
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates normalized internal fields and injects the API key into the item.
- **Configuration choices:**  
  Assigns:
  - `tipoDeNegocio` from `{{$json["Tipo de Negócio"]}}`
  - `cidade` from `{{$json.Cidade}}`
  - `api_key` as a static string placeholder: `YOUR_API_KEY_HERE`
  - `alwaysOutputData` enabled
- **Key expressions or variables used:**  
  - `{{$json["Tipo de Negócio"]}}`
  - `{{$json.Cidade}}`
- **Input and output connections:**  
  - Input ← `Forms Trigger`
  - Output → `Google Maps Text Search`
- **Version-specific requirements:**  
  Uses `Set` node version `3.4`.
- **Edge cases or potential failure types:**  
  - If field labels are changed in the form, these expressions will break
  - If `api_key` is not replaced with a valid key, all Google API requests fail
  - `alwaysOutputData` helps preserve output even if source fields are missing, but the resulting query may still be invalid
- **Sub-workflow reference:**  
  None

---

## 2.2 Google Maps Search & Pagination

### Overview
This block queries the Google Places Text Search API using the business category and city provided in the form. It automatically paginates through additional pages of Google results using `next_page_token`.

### Nodes Involved
- Google Maps Text Search

### Node Details

#### Google Maps Text Search
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Places Text Search endpoint to discover businesses matching the search phrase.
- **Configuration choices:**  
  - Method is implicitly GET through query parameters
  - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`
  - Query parameters:
    - `query` = `{{$json.tipoDeNegocio}} em {{$json.cidade}}`
    - `key` = `{{$('Format Search Query').item.json.api_key}}`
  - Pagination enabled:
    - Adds `pagetoken` from `{{$response.body.next_page_token}}`
    - Wait interval: `5000 ms`
    - Stop when `!$response.body.next_page_token`
- **Key expressions or variables used:**  
  - `{{$json.tipoDeNegocio}} em {{$json.cidade}}`
  - `{{$('Format Search Query').item.json.api_key}}`
  - `{{$response.body.next_page_token}}`
  - `{{ !$response.body.next_page_token }}`
- **Input and output connections:**  
  - Input ← `Format Search Query`
  - Outputs → `Merge Original & Details`, `Split Out`
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion 4.3`, which supports the configured pagination pattern.
- **Edge cases or potential failure types:**  
  - Invalid API key or billing disabled on Google Cloud
  - Google Places API not enabled
  - Quota exceeded or rate limiting
  - Pagination token not yet ready; the 5-second interval is important because Google next-page tokens often need a short delay before becoming valid
  - Broad searches can return many repetitive chain locations
  - The node note says “20 leads”, but pagination can produce more than 20 depending on API behavior
- **Sub-workflow reference:**  
  None

---

## 2.3 Per-Place Detail Enrichment

### Overview
This block converts each search result into an individual item and retrieves a smaller, more targeted details payload for each place. It enriches the lead record with phone and website fields, which are not always reliably present in the text search response.

### Nodes Involved
- Split Out
- Fetch Place Details

### Node Details

#### Split Out
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the `results` array from each Google Maps search response into one item per place.
- **Configuration choices:**  
  - Field to split: `results`
  - Includes all other fields alongside each split item
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input ← `Google Maps Text Search`
  - Output → `Fetch Place Details`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - If `results` is missing or empty, no detail lookups will occur
  - If the Google search response shape changes, splitting may fail or produce empty items
- **Sub-workflow reference:**  
  None

#### Fetch Place Details
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries Google Places Details API for each place ID to get contact-enrichment fields.
- **Configuration choices:**  
  - URL: `https://maps.googleapis.com/maps/api/place/details/json`
  - Query parameters:
    - `place_id` = `{{$json.results.place_id}}`
    - `fields` = `name,formatted_phone_number,website`
    - `key` = `{{$('Format Search Query').item.json.api_key}}`
  - Only selected fields are requested, which minimizes payload size and avoids unnecessary billing dimensions
- **Key expressions or variables used:**  
  - `{{$json.results.place_id}}`
  - `{{$('Format Search Query').item.json.api_key}}`
- **Input and output connections:**  
  - Input ← `Split Out`
  - Output → `Merge Original & Details`
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion 4.3`.
- **Edge cases or potential failure types:**  
  - Missing `place_id`
  - Invalid API key or quota exhaustion
  - Some places do not have phone or website data
  - Some details responses may contain only a `name`, which later affects validation
  - Network errors or intermittent Google API failures
- **Sub-workflow reference:**  
  None

---

## 2.4 Merge, Aggregation & Data Cleaning

### Overview
This block reunifies discovery data and enrichment data into one consolidated processing stream. After aggregation, a Code node reconstructs the dataset, normalizes place names, merges detail records into the original search results, and outputs a flattened SDR-friendly schema.

### Nodes Involved
- Merge Original & Details
- Aggregate
- Data Cleaning Logic

### Node Details

#### Merge Original & Details
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the original text search output stream with the details stream before aggregation.
- **Configuration choices:**  
  The node uses default merge settings. In this workflow, it effectively funnels both branches into a shared downstream path.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Inputs ← `Google Maps Text Search`, `Fetch Place Details`
  - Output → `Aggregate`
- **Version-specific requirements:**  
  Uses Merge `typeVersion 3.2`.
- **Edge cases or potential failure types:**  
  - Default merge behavior can be misunderstood; depending on n8n version and execution semantics, merge timing/order should be tested carefully
  - As the two branches have very different payload shapes, downstream logic must know how to reconstruct both datasets
- **Sub-workflow reference:**  
  None

#### Aggregate
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Collects all incoming items into a single aggregated structure so the Code node can inspect the full dataset in one execution.
- **Configuration choices:**  
  - Aggregate mode: `aggregateAllItemData`
  - Includes specified fields mode, though the actual behavior here is to gather item data for downstream processing
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input ← `Merge Original & Details`
  - Output → `Data Cleaning Logic`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Large result sets may increase memory usage
  - If the merged input stream is incomplete, the code logic will produce partial output
- **Sub-workflow reference:**  
  None

#### Data Cleaning Logic
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation node that rebuilds the lead dataset, deduplicates by normalized name key, and outputs a flat lead object per business.
- **Configuration choices:**  
  The code:
  1. Loads all items from `Google Maps Text Search`
  2. Extracts and flattens all `results`
  3. Defines a `normalizar()` helper to lowercase, remove accents, and strip non-alphanumeric characters
  4. Iterates over aggregated details data to build `detailsMap`
  5. Uses normalized `name` as the matching key
  6. Merges original result data with detail data
  7. Outputs fields:
     - `nome`
     - `tipo`
     - `endereco`
     - `latitude`
     - `longitude`
     - `aberto_agora`
     - `status`
     - `avaliacao`
     - `total_avaliacoes`
     - `telefone`
     - `site`
- **Key expressions or variables used:**  
  - `$items('Google Maps Text Search')`
  - `($json.data || [])`
- **Input and output connections:**  
  - Input ← `Aggregate`
  - Output → `Validate Lead Quality`
- **Version-specific requirements:**  
  Uses Code node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - Matching uses only normalized name, not name + address, although the comment says “nome + endereço”; this can incorrectly collapse multiple branches of the same chain
  - If `Aggregate` output format differs from what the code expects, `($json.data || [])` may not contain the anticipated structure
  - If Google search returns duplicate chain names with different addresses, the wrong phone/website can be assigned
  - Missing fields are tolerated with optional chaining, but may produce null or false defaults
  - The `aberto_agora` field defaults to `false` when missing, which may misrepresent “unknown” as “closed”
- **Sub-workflow reference:**  
  None

---

## 2.5 Lead Validation & Storage

### Overview
This block ensures only leads with at least a phone number are stored. Qualified leads are then appended to a predefined Google Sheet using explicit column mapping.

### Nodes Involved
- Validate Lead Quality
- Log Lead to Google Sheets

### Node Details

#### Validate Lead Quality
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters output based on presence of phone data.
- **Configuration choices:**  
  - Condition type: string exists
  - Left value: `{{$json.telefone}}`
  - Uses strict validation settings with AND combinator
- **Key expressions or variables used:**  
  - `{{$json.telefone}}`
- **Input and output connections:**  
  - Input ← `Data Cleaning Logic`
  - True output → `Log Lead to Google Sheets`
  - False output not connected
- **Version-specific requirements:**  
  Uses If node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - “Exists” checks field presence but not true quality; blank strings or malformed phone numbers may still need stronger validation
  - Leads without phone numbers are silently dropped because the false branch is unused
- **Sub-workflow reference:**  
  None

#### Log Lead to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends validated lead records into a target worksheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: document named `base`
  - Sheet/tab: `SRD`
  - OAuth2 credentials are used
  - Explicit field mapping:
    - `name` ← `{{$json.nome}}`
    - `site` ← `{{$json.site}}`
    - `types` ← `{{$json.tipo}}`
    - `rating` ← `{{$json.avaliacao}}`
    - `business_status` ← `{{$json.status}}`
    - `formatted_address` ← `{{$json.endereco}}`
    - `location.lat / lng` ← `{{$json.latitude}},{{$json.longitude}}`
    - `user_ratings_total` ← `{{$json.total_avaliacoes}}`
    - `formatted_phone_number` ← `{{$json.telefone}}`
    - `opening_hours.open_now` ← `{{$json.aberto_agora}}`
- **Key expressions or variables used:**  
  All mappings above use expressions from the current item.
- **Input and output connections:**  
  - Input ← `Validate Lead Quality`
  - No downstream connection
- **Version-specific requirements:**  
  Uses Google Sheets node `typeVersion 4.7`.
- **Edge cases or potential failure types:**  
  - OAuth2 credential expiration or insufficient sheet permissions
  - Sheet renamed, deleted, or column structure changed
  - Type mismatch if Google Sheets formatting is strict
  - Duplicate leads are appended without deduplication at storage level
- **Sub-workflow reference:**  
  None

---

## 2.6 Documentation & In-Canvas Notes

### Overview
These sticky notes document the workflow’s sections and purpose directly in the n8n canvas. They do not affect execution but are important for maintainability and context.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation label for the first block.
- **Configuration choices:**  
  Content: `## TRIGGER & INPUT PREP`
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None; non-executable
- **Sub-workflow reference:**  
  None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `## GOOGLE MAPS ENRICHMENT`
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `## DATA PROCESSING & LOGIC`
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `## VALIDATION & STORAGE`
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Large descriptive note containing:
  - Pipeline title
  - Objective
  - Functional explanation
  - Tech stack
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Forms Trigger | n8n-nodes-base.formTrigger | Entry form for business type and city |  | Format Search Query | ## TRIGGER & INPUT PREP |
| Format Search Query | n8n-nodes-base.set | Normalize form inputs and store Google API key | Forms Trigger | Google Maps Text Search | ## TRIGGER & INPUT PREP |
| Google Maps Text Search | n8n-nodes-base.httpRequest | Query Google Places Text Search with pagination | Format Search Query | Merge Original & Details; Split Out | ## GOOGLE MAPS ENRICHMENT |
| Split Out | n8n-nodes-base.splitOut | Split search results array into one item per place | Google Maps Text Search | Fetch Place Details | ## GOOGLE MAPS ENRICHMENT |
| Fetch Place Details | n8n-nodes-base.httpRequest | Fetch phone and website for each place ID | Split Out | Merge Original & Details | ## GOOGLE MAPS ENRICHMENT |
| Merge Original & Details | n8n-nodes-base.merge | Combine search and detail branches before aggregation | Google Maps Text Search; Fetch Place Details | Aggregate | ## DATA PROCESSING & LOGIC |
| Aggregate | n8n-nodes-base.aggregate | Gather incoming items for whole-dataset processing | Merge Original & Details | Data Cleaning Logic | ## DATA PROCESSING & LOGIC |
| Data Cleaning Logic | n8n-nodes-base.code | Normalize, merge, flatten, and prepare lead records | Aggregate | Validate Lead Quality | ## DATA PROCESSING & LOGIC |
| Validate Lead Quality | n8n-nodes-base.if | Keep only leads with phone data | Data Cleaning Logic | Log Lead to Google Sheets | ## VALIDATION & STORAGE |
| Log Lead to Google Sheets | n8n-nodes-base.googleSheets | Append qualified leads to Google Sheets | Validate Lead Quality |  | ## VALIDATION & STORAGE |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Google Maps Lead Gen`.
   - Keep the timezone aligned with your environment; the source workflow uses `America/Sao_Paulo`.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Title: `SDR`
   - Add two form fields:
     1. `Tipo de Negócio`
     2. `Cidade`
   - This will be the only entry point of the workflow.

3. **Add a Set node after the form**
   - Node type: **Set**
   - Name it: `Format Search Query`
   - Add the following fields:
     - `tipoDeNegocio` as string = `{{$json["Tipo de Negócio"]}}`
     - `cidade` as string = `{{$json.Cidade}}`
     - `api_key` as string = `YOUR_API_KEY_HERE`
   - Enable **Always Output Data**
   - Connect:
     - `Forms Trigger` → `Format Search Query`

4. **Replace the placeholder API key**
   - Edit the `api_key` value and replace `YOUR_API_KEY_HERE` with a valid Google Places API key.
   - Ensure in Google Cloud:
     - Billing is enabled
     - Places API / Maps Places API is enabled
     - The key is allowed to call Places endpoints

5. **Add the Google Places Text Search request**
   - Node type: **HTTP Request**
   - Name: `Google Maps Text Search`
   - Method: GET
   - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`
   - Enable query parameters:
     - `query` = `{{$json.tipoDeNegocio}} em {{$json.cidade}}`
     - `key` = `{{$('Format Search Query').item.json.api_key}}`
   - Enable pagination:
     - Add pagination parameter `pagetoken` = `{{$response.body.next_page_token}}`
     - Set request interval to `5000 ms`
     - Completion expression = `{{ !$response.body.next_page_token }}`
   - Connect:
     - `Format Search Query` → `Google Maps Text Search`

6. **Add a Split Out node**
   - Node type: **Split Out**
   - Name: `Split Out`
   - Field to split out: `results`
   - Include: **All Other Fields**
   - Connect:
     - `Google Maps Text Search` → `Split Out`

7. **Add the Google Places Details request**
   - Node type: **HTTP Request**
   - Name: `Fetch Place Details`
   - Method: GET
   - URL: `https://maps.googleapis.com/maps/api/place/details/json`
   - Add query parameters:
     - `place_id` = `{{$json.results.place_id}}`
     - `fields` = `name,formatted_phone_number,website`
     - `key` = `{{$('Format Search Query').item.json.api_key}}`
   - Connect:
     - `Split Out` → `Fetch Place Details`

8. **Add a Merge node**
   - Node type: **Merge**
   - Name: `Merge Original & Details`
   - Keep default settings unless your n8n version requires explicit mode selection.
   - Connect both branches into it:
     - `Google Maps Text Search` → `Merge Original & Details`
     - `Fetch Place Details` → `Merge Original & Details`

9. **Add an Aggregate node**
   - Node type: **Aggregate**
   - Name: `Aggregate`
   - Aggregate mode: `Aggregate All Item Data`
   - Keep other options at default unless your instance requires explicit field inclusion rules
   - Connect:
     - `Merge Original & Details` → `Aggregate`

10. **Add a Code node**
    - Node type: **Code**
    - Name: `Data Cleaning Logic`
    - Language: JavaScript
    - Paste logic equivalent to this behavior:
      - Load all items from `Google Maps Text Search`
      - Flatten all `results`
      - Normalize names by:
        - lowercasing
        - removing accents
        - removing non-alphanumeric characters
      - Build a detail lookup from aggregated detail results
      - Merge base place data with details
      - Output one item per lead with:
        - `nome`
        - `tipo`
        - `endereco`
        - `latitude`
        - `longitude`
        - `aberto_agora`
        - `status`
        - `avaliacao`
        - `total_avaliacoes`
        - `telefone`
        - `site`
    - Use the same expressions/API helpers as in the original:
      - `$items('Google Maps Text Search')`
      - aggregated input data from the current item
    - Connect:
      - `Aggregate` → `Data Cleaning Logic`

11. **Recommended improvement while recreating**
    - The original code comments mention deduplication using name + address, but the implemented key uses only normalized name.
    - To avoid cross-branch collisions for chain businesses, prefer a composite key such as normalized `name + formatted_address` or, even better, `place_id` if preserved throughout.

12. **Add an If node**
    - Node type: **If**
    - Name: `Validate Lead Quality`
    - Condition:
      - Check whether `{{$json.telefone}}` **exists**
    - Use the true branch only
    - Connect:
      - `Data Cleaning Logic` → `Validate Lead Quality`

13. **Prepare the destination Google Sheet**
    - Create a Google Spreadsheet, e.g. named `base`
    - Create a worksheet/tab, e.g. `SRD`
    - Add these columns in the header row:
      - `types`
      - `name`
      - `formatted_phone_number`
      - `formatted_address`
      - `location.lat / lng`
      - `opening_hours.open_now`
      - `business_status`
      - `rating`
      - `user_ratings_total`
      - `site`

14. **Create Google Sheets credentials in n8n**
    - Credential type: **Google Sheets OAuth2 API**
    - Authenticate with a Google account that has edit access to the spreadsheet
    - Verify n8n can list and write to the document

15. **Add the Google Sheets node**
    - Node type: **Google Sheets**
    - Name: `Log Lead to Google Sheets`
    - Operation: `Append`
    - Select the spreadsheet document
    - Select the `SRD` sheet
    - Mapping mode: define fields manually
    - Map:
      - `name` = `{{$json.nome}}`
      - `site` = `{{$json.site}}`
      - `types` = `{{$json.tipo}}`
      - `rating` = `{{$json.avaliacao}}`
      - `business_status` = `{{$json.status}}`
      - `formatted_address` = `{{$json.endereco}}`
      - `location.lat / lng` = `{{$json.latitude}},{{$json.longitude}}`
      - `user_ratings_total` = `{{$json.total_avaliacoes}}`
      - `formatted_phone_number` = `{{$json.telefone}}`
      - `opening_hours.open_now` = `{{$json.aberto_agora}}`
    - Connect:
      - True branch of `Validate Lead Quality` → `Log Lead to Google Sheets`

16. **Optionally add visual sticky notes**
    - Add sticky notes for maintainability:
      - `## TRIGGER & INPUT PREP`
      - `## GOOGLE MAPS ENRICHMENT`
      - `## DATA PROCESSING & LOGIC`
      - `## VALIDATION & STORAGE`
      - A larger note describing the business purpose and tech stack

17. **Test the workflow**
    - Submit sample values such as:
      - `Tipo de Negócio`: `farmacia`
      - `Cidade`: `Fortaleza`
    - Confirm:
      - The text search returns multiple pages
      - `Split Out` creates one item per place
      - Details requests return phone/website where available
      - The code node outputs flattened lead records
      - Only records with `telefone` reach Google Sheets

18. **Validate production-readiness**
    - Check Google API quotas and billing
    - Consider adding:
      - error handling branch
      - deduplication before Google Sheets append
      - stronger phone validation
      - logging of rejected leads
      - preservation of `place_id` for future updates/upserts

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SDR Google Maps Lead Pipeline | Workflow branding/title shown in the canvas note |
| Objective: high-efficiency B2B lead generation engine for SDR teams using Google Maps, n8n, and Google Sheets | Canvas documentation |
| Trigger & Prep: starts via n8n Form and Set node prepares inputs/environment variables | Canvas documentation |
| Discovery & Enrichment: uses Google Maps Text Search with auto-pagination while `next_page_token` exists | Canvas documentation |
| Granular Fetch: Split Out sends each result to Place Details lookup for phone number and website | Canvas documentation |
| Data Intelligence: custom JavaScript normalizes strings, removes duplicates, and prepares output schema | Canvas documentation |
| Quality Gate: If node only keeps leads with a valid phone number | Canvas documentation |
| Tech Stack: Source = Google Places API, Engine = n8n Automation, Storage = Google Sheets | Canvas documentation |

## Additional implementation notes

- There is **one entry point only**: `Forms Trigger`.
- There are **no sub-workflows** and no Execute Workflow nodes.
- The workflow is currently **inactive** in the provided export.
- The code node’s implemented deduplication logic does **not fully match its comment**. The comment refers to a composite key using name and address, but the actual key uses only normalized name.
- The Google Maps Text Search note says **“20 leads”**, but the configured pagination can return multiple pages, so the real number of processed results may exceed that.