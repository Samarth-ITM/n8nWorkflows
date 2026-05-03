Generate weekly supply chain OTIF reports and AI analysis with Notion and GPT-4o

https://n8nworkflows.xyz/workflows/generate-weekly-supply-chain-otif-reports-and-ai-analysis-with-notion-and-gpt-4o-13944


# Generate weekly supply chain OTIF reports and AI analysis with Notion and GPT-4o

# 1. Workflow Overview

This workflow generates a **weekly OTIF (On-Time In-Full) logistics performance report** from shipment records, enriches the results with **AI-written commentary**, and stores both the KPI data and analysis in **Notion**.

Its main use case is operational or management reporting for retail supply chains, especially where shipment milestone punctuality needs to be tracked week by week and summarised for decision-makers.

## 1.1 Scheduled Input Reception

The workflow starts on a **weekly schedule** and retrieves raw shipment records from a data source represented here by an n8n **Data Table** node.

## 1.2 Weekly KPI Aggregation

It groups all shipment rows by ISO-style week start, then computes weekly KPIs such as:

- total shipments
- on-time deliveries
- late shipments
- average lead time in days and hours
- on-time rates for transmission, loading, airport, landing, and delivery checkpoints

## 1.3 Weekly AI Analysis

For each aggregated weekly record, an **AI Agent** generates a short business analysis describing performance, the weakest checkpoint, and one recommendation.

## 1.4 Weekly Reporting to Notion

Two Notion outputs are produced for each week:

- a KPI row in the **Daily OTIF Summary** database
- an AI commentary row in the **AI Analysis** database

## 1.5 Global Cross-Week Summary

After all weekly records are aggregated, a second code node builds a consolidated text prompt containing all weekly indicators. A second AI Agent uses that prompt to generate a **global management summary** across all weeks.

## 1.6 Global Notion Update

The generated overall summary is written back to a dedicated Notion page row labelled **Overall Performance Summary**.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Trigger and Collect Shipment Records

### Overview

This block launches the workflow automatically every week and fetches all shipment records from the configured source. It is the entry point for the entire reporting process.

### Nodes Involved

- Weekly Trigger
- Collect Shipments from TMS & WMS

### Node Details

#### Weekly Trigger

- **Type and role:** `n8n-nodes-base.scheduleTrigger`; starts the workflow on a weekly cadence.
- **Configuration choices:** Configured with an interval rule based on `weeks`, meaning the workflow runs once per week.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Collect Shipments from TMS & WMS**.
- **Version-specific requirements:** Uses type version `1.3`.
- **Edge cases or failures:**
  - Workflow may not run if inactive.
  - Schedule timing depends on instance timezone.
  - If the workflow execution overlaps with another long-running run, operational contention is possible.
- **Sub-workflow reference:** None.

#### Collect Shipments from TMS & WMS

- **Type and role:** `n8n-nodes-base.dataTable`; retrieves all shipment rows from an n8n Data Table.
- **Configuration choices:**
  - Operation: `get`
  - Return all rows: enabled
  - Requires a selected Data Table ID
- **Key expressions or variables used:** None in the current node configuration.
- **Input and output connections:** Receives input from **Weekly Trigger**; outputs to **Aggregate by Week**.
- **Version-specific requirements:** Uses type version `1.1`.
- **Edge cases or failures:**
  - Empty table produces no shipment rows downstream.
  - Wrong or missing table ID prevents retrieval.
  - If adapted to an external source later, pagination, authentication, or rate limiting may become relevant.
- **Sub-workflow reference:** None.

---

## 2.2 Block: Aggregate Shipments and Compute KPIs

### Overview

This block transforms raw shipment records into one record per week. It calculates all operational KPIs used later by the AI analysis and Notion reporting branches.

### Nodes Involved

- Aggregate by Week

### Node Details

#### Aggregate by Week

- **Type and role:** `n8n-nodes-base.code`; custom JavaScript aggregation logic.
- **Configuration choices:**
  - Reads all incoming shipment items with `$input.all()`.
  - Groups records by Monday-based week key derived from `Order_Time`.
  - Calculates:
    - `Name` as `Week <number>`
    - `weekStart`, `weekEnd`
    - `totalShipments`
    - `onTimeDeliveries`
    - `avgLeadTimeDays`
    - `avgLeadTimeHrs`
    - checkpoint percentages:
      - `transmissionOnTime`
      - `loadingOnTime`
      - `airportOnTime`
      - `landingOnTime`
      - `deliveryOnTime`
    - `lateShipments`
  - Sorts results ascending by `weekStart`.
- **Key expressions or variables used:**
  - `s.Order_Time`
  - `r.Delivery_OnTime`
  - `r.LT_Days`
  - `r.LT_Hours`
  - `r.Transmission_OnTime`
  - `r.Loading_OnTime`
  - `r.Airport_OnTime`
  - `r.Landing_OnTime`
- **Input and output connections:** Receives input from **Collect Shipments from TMS & WMS**; outputs to:
  - **AI Agent Weekly Performance Summary**
  - **Prepare Global Summary Prompt with Indicators**
- **Version-specific requirements:** Uses code node type version `2`.
- **Edge cases or failures:**
  - If no shipments are received, the node returns a single item with `{ error: 'No shipments found', count: 0 }`. Downstream nodes are not designed explicitly for that shape, so later expressions may fail or produce poor summaries.
  - Invalid or missing `Order_Time` can create invalid date values.
  - Missing numeric fields are partly tolerated with `(value || 0)` logic for lead times, but boolean metrics assume proper `true` values.
  - Division-by-zero is avoided in practice because KPI calculation only occurs for populated weekly groups, but malformed records could still create inconsistent outputs.
  - Week number logic is custom and may differ slightly from strict ISO week expectations in edge calendar cases.
- **Sub-workflow reference:** None.

---

## 2.3 Block: AI Agent Generates a Per-Week Performance Analysis

### Overview

This block sends each weekly KPI object to an AI Agent, which produces a concise operational assessment. A dedicated OpenAI chat model is attached as the language model backend.

### Nodes Involved

- OpenAI Chat Model
- AI Agent Weekly Performance Summary

### Node Details

#### OpenAI Chat Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider for the weekly AI agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: `300`
  - Temperature: `0.3`
- **Key expressions or variables used:** None directly in this node.
- **Input and output connections:** Connected via `ai_languageModel` to **AI Agent Weekly Performance Summary**.
- **Version-specific requirements:** Uses type version `1.2`.
- **Edge cases or failures:**
  - Missing or invalid OpenAI credentials.
  - Model access restrictions on the account.
  - Token or rate-limit issues.
  - Temporary API failures.
- **Sub-workflow reference:** None.

#### AI Agent Weekly Performance Summary

- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; generates a textual weekly logistics analysis.
- **Configuration choices:**
  - Prompt type: defined directly in the node.
  - System message instructs the agent to act as a supply chain OTIF analyst, be concise and actionable, and use British spelling.
  - The main prompt injects KPI fields from the current weekly JSON item and requests:
    - overall performance assessment
    - weakest checkpoint
    - one specific recommendation
    - 3–4 concise sentences
- **Key expressions or variables used:**
  - `{{ $json.Name }}`
  - `{{ $json.weekStart }}`
  - `{{ $json.weekEnd }}`
  - `{{ $json.totalShipments }}`
  - `{{ $json.onTimeDeliveries }}`
  - `{{ $json.lateShipments }}`
  - `{{ Math.round($json.onTimeDeliveries / $json.totalShipments * 1000) / 10 }}`
  - `{{ $json.avgLeadTimeDays }}`
  - `{{ $json.avgLeadTimeHrs }}`
  - checkpoint percentages from `$json`
- **Input and output connections:**
  - Main input from **Aggregate by Week**
  - AI language model input from **OpenAI Chat Model**
  - Main outputs to:
    - **Fill the report**
    - **Create Weekly Performance Card**
- **Version-specific requirements:** Uses type version `1.7`.
- **Edge cases or failures:**
  - If upstream data is missing expected KPI fields, the prompt may render incorrectly.
  - If `totalShipments` is zero or undefined, the OTIF percentage expression can become invalid.
  - LLM output is non-deterministic, though reduced by low temperature.
  - Output format depends on the agent’s standard `output` field; downstream nodes assume this field exists.
- **Sub-workflow reference:** None.

---

## 2.4 Block: Push Weekly KPIs and AI Analysis Cards to Notion

### Overview

This block stores the weekly results in Notion. One node writes KPI metrics to a scorecard database, while another writes the AI-generated weekly commentary to a separate analysis database.

### Nodes Involved

- Fill the report
- Create Weekly Performance Card

### Node Details

#### Fill the report

- **Type and role:** `n8n-nodes-base.notion`; creates a page in a Notion database for weekly KPI reporting.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: create
  - Title and `Name|title` come from `Aggregate by Week`
  - Icon: truck emoji `🚚`
  - Writes numeric fields for all calculated KPI metrics
  - Writes a date range using `weekStart` and `weekEnd`
  - Includes an `AI Analysis|rich_text` property, but it is left empty in the current configuration
  - Database target is the Notion database labelled `Daily OTIF Summary`
- **Key expressions or variables used:**
  - `{{ $('Aggregate by Week').item.json.Name }}`
  - `{{ $('Aggregate by Week').item.json.airportOnTime }}`
  - `{{ $('Aggregate by Week').item.json.avgLeadTimeDays }}`
  - `{{ $('Aggregate by Week').item.json.avgLeadTimeHrs }}`
  - `{{ $('Aggregate by Week').item.json.weekEnd }}`
  - `{{ $('Aggregate by Week').item.json.weekStart }}`
  - `{{ $('Aggregate by Week').item.json.deliveryOnTime }}`
  - `{{ $('Aggregate by Week').item.json.landingOnTime }}`
  - `{{ $('Aggregate by Week').item.json.lateShipments }}`
  - `{{ $('Aggregate by Week').item.json.loadingOnTime }}`
  - `{{ $('Aggregate by Week').item.json.onTimeDeliveries }}`
  - `{{ $('Aggregate by Week').item.json.totalShipments }}`
  - `{{ $('Aggregate by Week').item.json.transmissionOnTime }}`
- **Input and output connections:** Receives input from **AI Agent Weekly Performance Summary**; no downstream connections.
- **Version-specific requirements:** Uses Notion node type version `2.2`.
- **Edge cases or failures:**
  - Missing Notion credentials or insufficient database permissions.
  - Database schema mismatch: every configured property name must exist exactly in Notion.
  - Cross-node `.item` references can fail if item linking changes unexpectedly.
  - The AI Analysis property is present but not populated, which may be intentional or an incomplete design.
- **Sub-workflow reference:** None.

#### Create Weekly Performance Card

- **Type and role:** `n8n-nodes-base.notion`; creates a page in a second Notion database for AI commentary.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: create
  - Icon: chart emoji `📊`
  - Database target labelled `AI Analysis`
  - Writes:
    - `Name|title` from aggregated week name
    - `Weekly Analysis|rich_text` from AI output
    - `Type|select` as `Weekly Analysis`
    - `Updated|date` with current execution date generated by Notion node behavior for date property when included without explicit value
- **Key expressions or variables used:**
  - `{{ $('Aggregate by Week').item.json.Name }}`
  - `{{ $json.output }}`
- **Input and output connections:** Receives input from **AI Agent Weekly Performance Summary**; no downstream connections.
- **Version-specific requirements:** Uses Notion node type version `2.2`.
- **Edge cases or failures:**
  - Same Notion auth and schema concerns as above.
  - If the AI agent output field is absent, the rich text field will be blank or error.
  - The `Type` select option must already exist or be supported by the Notion API configuration.
- **Sub-workflow reference:** None.

---

## 2.5 Block: AI Agent Generates a Global Cross-Week Performance Summary

### Overview

This block consolidates all weekly KPI rows into one textual management prompt, then asks a second AI Agent to create a cross-week summary with trends, bottlenecks, best/worst weeks, and recommendations.

### Nodes Involved

- Prepare Global Summary Prompt with Indicators
- OpenAI Chat Model Global
- AI Agent Global Performance Summary

### Node Details

#### Prepare Global Summary Prompt with Indicators

- **Type and role:** `n8n-nodes-base.code`; builds a single prompt string covering every aggregated week.
- **Configuration choices:**
  - Reads all weekly items using `$input.all()`
  - Sorts by `weekStart`
  - Creates a `globalPrompt` text block containing weekly shipment counts, OTIF, lead times, and checkpoint rates
  - Returns a single item with `{ globalPrompt: summary }`
- **Key expressions or variables used:**
  - `w.weekStart`
  - `w.weekEnd`
  - `w.onTimeDeliveries`
  - `w.totalShipments`
  - `w.lateShipments`
  - `w.avgLeadTimeDays`
  - `w.transmissionOnTime`
  - `w.loadingOnTime`
  - `w.airportOnTime`
  - `w.landingOnTime`
  - `w.deliveryOnTime`
- **Input and output connections:** Receives input from **Aggregate by Week**; outputs to **AI Agent Global Performance Summary**.
- **Version-specific requirements:** Uses code node type version `2`.
- **Edge cases or failures:**
  - If upstream contains an error item instead of KPI items, the generated prompt will be incomplete or malformed.
  - Division-based OTIF calculation assumes `totalShipments > 0`.
  - Very long historical data could make the prompt large; token growth may matter if many weeks are included.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model Global

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider for the global AI agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: `600`
  - Temperature: `0.3`
- **Key expressions or variables used:** None directly.
- **Input and output connections:** Connected via `ai_languageModel` to **AI Agent Global Performance Summary**.
- **Version-specific requirements:** Uses type version `1.2`.
- **Edge cases or failures:**
  - Same OpenAI credential, access, and rate-limit risks as the weekly model node.
  - If prompts become too long, truncation or quality degradation may occur.
- **Sub-workflow reference:** None.

#### AI Agent Global Performance Summary

- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; produces a multi-week management summary.
- **Configuration choices:**
  - Prompt type: defined directly in the node.
  - Prompt begins with `{{ $json.globalPrompt }}`
  - Explicitly asks for:
    1. overall trend
    2. most consistent bottleneck checkpoint
    3. best and worst weeks with reasons
    4. two recommendations
    5. outlook
  - Output format requested as concise bullet points without markdown headers
  - System message sets role, scope, and British spelling
- **Key expressions or variables used:**
  - `{{ $json.globalPrompt }}`
- **Input and output connections:**
  - Main input from **Prepare Global Summary Prompt with Indicators**
  - AI language model input from **OpenAI Chat Model Global**
  - Main output to **Update Global Performance Summary**
- **Version-specific requirements:** Uses type version `1.7`.
- **Edge cases or failures:**
  - If the summary prompt is empty, the agent output will be low quality.
  - Output shape must contain `output` for downstream Notion update.
  - Large weekly histories can increase latency and cost.
- **Sub-workflow reference:** None.

---

## 2.6 Block: Update the Global Performance Summary Card in Notion

### Overview

This block writes the final cross-week AI analysis into a dedicated Notion page. It acts as the management-facing rollup of the whole workflow.

### Nodes Involved

- Update Global Performance Summary

### Node Details

#### Update Global Performance Summary

- **Type and role:** `n8n-nodes-base.notion`; updates an existing Notion database page rather than creating a new one.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `update`
  - Target page selected by explicit `pageId`
  - Sets:
    - `Name|title` = `Overall Performance Summary`
    - `Weekly Analysis|rich_text` = AI output
    - `Type|select` = `Global Summary`
    - `Updated|date`
- **Key expressions or variables used:**
  - `{{ $json.output }}`
- **Input and output connections:** Receives input from **AI Agent Global Performance Summary**; no downstream node.
- **Version-specific requirements:** Uses Notion node type version `2.2`.
- **Edge cases or failures:**
  - `pageId` must point to an existing page in the correct database.
  - Notion permissions must allow update access.
  - Property names and types must match the database schema exactly.
  - If the page is deleted or moved, updates will fail.
- **Sub-workflow reference:** None.

---

## 2.7 Non-Executable Documentation Nodes

### Overview

These nodes are visual annotations only. They explain setup, process stages, and provide a video link.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7

### Node Details

All sticky notes are of type `n8n-nodes-base.stickyNote` and have no execution role, inputs, or outputs.

#### Sticky Note

- General overview of the whole workflow.
- Includes setup checklist:
  - connect data source
  - add OpenAI credentials
  - add Notion credentials
  - verify Notion database IDs
- Includes customisation notes:
  - adjust AI prompts
  - modify KPI logic in aggregation code

#### Sticky Note1

- Section label: `1. Trigger and collect shipment records`

#### Sticky Note2

- Section label: `2. Aggregate shipments and compute KPIs`

#### Sticky Note3

- Section label: `3. AI Agent generates a per-week performance analysis`

#### Sticky Note4

- Section label: `4. Push weekly KPIs and AI analysis cards to Notion`

#### Sticky Note5

- Section label: `5. AI Agent generates a global cross-week performance summary`

#### Sticky Note6

- Section label: `6. Update the global performance summary card in Notion`

#### Sticky Note7

- Contains a video link:
  - `https://www.youtube.com/watch?v=tOT8XhQ7eB8`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | n8n-nodes-base.scheduleTrigger | Weekly workflow entry point |  | Collect Shipments from TMS & WMS | ## Supply Chain OTIF Performance Scorecard for Retail Logistics<br>Automatically aggregate shipment data, compute weekly OTIF KPIs, generate AI-powered performance analysis, and push everything to a Notion dashboard.<br>### How it Works<br>1. **Collect** raw shipment records from your TMS/WMS data source.<br>2. **Aggregate** shipments by ISO week and compute KPIs (OTIF rate, lead times, checkpoint on-time rates).<br>3. **AI Weekly Analysis** generates a per-week performance comment using an AI Agent.<br>4. **Push to Notion** creates one row per week in the OTIF Summary database and one card per week in the AI Analysis database.<br>5. **AI Global Analysis** generates a cross-week summary with trends and recommendations.<br>6. **Update Global Card** pushes the overall summary to a dedicated Notion database row.<br>### Setup<br>- [ ] Connect your **data source** (DataTable, database, or API) to the "Collect Shipments" node<br>- [ ] Add your **OpenAI API Key** to both OpenAI Chat Model nodes<br>- [ ] Add your **Notion API credentials** to all Notion nodes<br>- [ ] Verify the **Notion database IDs** match your workspace<br>### Customisation<br>- Adjust the AI prompts to change the analysis style or language<br>- Modify the Aggregate by Week code to add custom KPI<br>## 1. Trigger and collect shipment records |
| Collect Shipments from TMS & WMS | n8n-nodes-base.dataTable | Fetch all shipment records | Weekly Trigger | Aggregate by Week | ## Supply Chain OTIF Performance Scorecard for Retail Logistics<br>Automatically aggregate shipment data, compute weekly OTIF KPIs, generate AI-powered performance analysis, and push everything to a Notion dashboard.<br>### How it Works<br>1. **Collect** raw shipment records from your TMS/WMS data source.<br>2. **Aggregate** shipments by ISO week and compute KPIs (OTIF rate, lead times, checkpoint on-time rates).<br>3. **AI Weekly Analysis** generates a per-week performance comment using an AI Agent.<br>4. **Push to Notion** creates one row per week in the OTIF Summary database and one card per week in the AI Analysis database.<br>5. **AI Global Analysis** generates a cross-week summary with trends and recommendations.<br>6. **Update Global Card** pushes the overall summary to a dedicated Notion database row.<br>### Setup<br>- [ ] Connect your **data source** (DataTable, database, or API) to the "Collect Shipments" node<br>- [ ] Add your **OpenAI API Key** to both OpenAI Chat Model nodes<br>- [ ] Add your **Notion API credentials** to all Notion nodes<br>- [ ] Verify the **Notion database IDs** match your workspace<br>### Customisation<br>- Adjust the AI prompts to change the analysis style or language<br>- Modify the Aggregate by Week code to add custom KPI<br>## 1. Trigger and collect shipment records |
| Aggregate by Week | n8n-nodes-base.code | Group shipments by week and compute KPIs | Collect Shipments from TMS & WMS | AI Agent Weekly Performance Summary; Prepare Global Summary Prompt with Indicators | ## Supply Chain OTIF Performance Scorecard for Retail Logistics<br>Automatically aggregate shipment data, compute weekly OTIF KPIs, generate AI-powered performance analysis, and push everything to a Notion dashboard.<br>### How it Works<br>1. **Collect** raw shipment records from your TMS/WMS data source.<br>2. **Aggregate** shipments by ISO week and compute KPIs (OTIF rate, lead times, checkpoint on-time rates).<br>3. **AI Weekly Analysis** generates a per-week performance comment using an AI Agent.<br>4. **Push to Notion** creates one row per week in the OTIF Summary database and one card per week in the AI Analysis database.<br>5. **AI Global Analysis** generates a cross-week summary with trends and recommendations.<br>6. **Update Global Card** pushes the overall summary to a dedicated Notion database row.<br>### Setup<br>- [ ] Connect your **data source** (DataTable, database, or API) to the "Collect Shipments" node<br>- [ ] Add your **OpenAI API Key** to both OpenAI Chat Model nodes<br>- [ ] Add your **Notion API credentials** to all Notion nodes<br>- [ ] Verify the **Notion database IDs** match your workspace<br>### Customisation<br>- Adjust the AI prompts to change the analysis style or language<br>- Modify the Aggregate by Week code to add custom KPI<br>## 2. Aggregate shipments and compute KPIs |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for weekly AI analysis |  | AI Agent Weekly Performance Summary | ## 3. AI Agent generates a per-week performance analysis |
| AI Agent Weekly Performance Summary | @n8n/n8n-nodes-langchain.agent | Generate weekly AI commentary | Aggregate by Week; OpenAI Chat Model | Fill the report; Create Weekly Performance Card | ## Supply Chain OTIF Performance Scorecard for Retail Logistics<br>Automatically aggregate shipment data, compute weekly OTIF KPIs, generate AI-powered performance analysis, and push everything to a Notion dashboard.<br>### How it Works<br>1. **Collect** raw shipment records from your TMS/WMS data source.<br>2. **Aggregate** shipments by ISO week and compute KPIs (OTIF rate, lead times, checkpoint on-time rates).<br>3. **AI Weekly Analysis** generates a per-week performance comment using an AI Agent.<br>4. **Push to Notion** creates one row per week in the OTIF Summary database and one card per week in the AI Analysis database.<br>5. **AI Global Analysis** generates a cross-week summary with trends and recommendations.<br>6. **Update Global Card** pushes the overall summary to a dedicated Notion database row.<br>### Setup<br>- [ ] Connect your **data source** (DataTable, database, or API) to the "Collect Shipments" node<br>- [ ] Add your **OpenAI API Key** to both OpenAI Chat Model nodes<br>- [ ] Add your **Notion API credentials** to all Notion nodes<br>- [ ] Verify the **Notion database IDs** match your workspace<br>### Customisation<br>- Adjust the AI prompts to change the analysis style or language<br>- Modify the Aggregate by Week code to add custom KPI<br>## 3. AI Agent generates a per-week performance analysis |
| Fill the report | n8n-nodes-base.notion | Create weekly KPI page in Notion OTIF database | AI Agent Weekly Performance Summary |  | ## 4. Push weekly KPIs and AI analysis cards to Notion |
| Create Weekly Performance Card | n8n-nodes-base.notion | Create weekly AI analysis page in Notion analysis database | AI Agent Weekly Performance Summary |  | ## 4. Push weekly KPIs and AI analysis cards to Notion |
| Prepare Global Summary Prompt with Indicators | n8n-nodes-base.code | Build a consolidated multi-week prompt | Aggregate by Week | AI Agent Global Performance Summary | ## Supply Chain OTIF Performance Scorecard for Retail Logistics<br>Automatically aggregate shipment data, compute weekly OTIF KPIs, generate AI-powered performance analysis, and push everything to a Notion dashboard.<br>### How it Works<br>1. **Collect** raw shipment records from your TMS/WMS data source.<br>2. **Aggregate** shipments by ISO week and compute KPIs (OTIF rate, lead times, checkpoint on-time rates).<br>3. **AI Weekly Analysis** generates a per-week performance comment using an AI Agent.<br>4. **Push to Notion** creates one row per week in the OTIF Summary database and one card per week in the AI Analysis database.<br>5. **AI Global Analysis** generates a cross-week summary with trends and recommendations.<br>6. **Update Global Card** pushes the overall summary to a dedicated Notion database row.<br>### Setup<br>- [ ] Connect your **data source** (DataTable, database, or API) to the "Collect Shipments" node<br>- [ ] Add your **OpenAI API Key** to both OpenAI Chat Model nodes<br>- [ ] Add your **Notion API credentials** to all Notion nodes<br>- [ ] Verify the **Notion database IDs** match your workspace<br>### Customisation<br>- Adjust the AI prompts to change the analysis style or language<br>- Modify the Aggregate by Week code to add custom KPI<br>## 5. AI Agent generates a global cross-week performance summary |
| OpenAI Chat Model Global | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for global AI summary |  | AI Agent Global Performance Summary | ## 5. AI Agent generates a global cross-week performance summary |
| AI Agent Global Performance Summary | @n8n/n8n-nodes-langchain.agent | Generate cross-week management summary | Prepare Global Summary Prompt with Indicators; OpenAI Chat Model Global | Update Global Performance Summary | ## Supply Chain OTIF Performance Scorecard for Retail Logistics<br>Automatically aggregate shipment data, compute weekly OTIF KPIs, generate AI-powered performance analysis, and push everything to a Notion dashboard.<br>### How it Works<br>1. **Collect** raw shipment records from your TMS/WMS data source.<br>2. **Aggregate** shipments by ISO week and compute KPIs (OTIF rate, lead times, checkpoint on-time rates).<br>3. **AI Weekly Analysis** generates a per-week performance comment using an AI Agent.<br>4. **Push to Notion** creates one row per week in the OTIF Summary database and one card per week in the AI Analysis database.<br>5. **AI Global Analysis** generates a cross-week summary with trends and recommendations.<br>6. **Update Global Card** pushes the overall summary to a dedicated Notion database row.<br>### Setup<br>- [ ] Connect your **data source** (DataTable, database, or API) to the "Collect Shipments" node<br>- [ ] Add your **OpenAI API Key** to both OpenAI Chat Model nodes<br>- [ ] Add your **Notion API credentials** to all Notion nodes<br>- [ ] Verify the **Notion database IDs** match your workspace<br>### Customisation<br>- Adjust the AI prompts to change the analysis style or language<br>- Modify the Aggregate by Week code to add custom KPI<br>## 5. AI Agent generates a global cross-week performance summary |
| Update Global Performance Summary | n8n-nodes-base.notion | Update a dedicated Notion page with overall AI summary | AI Agent Global Performance Summary |  | ## 6. Update the global performance summary card in Notion<br>## [Tutorial](https://www.youtube.com/watch?v=tOT8XhQ7eB8)<br>@[youtube](tOT8XhQ7eB8) |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual section label |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual section label |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual section label |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual section label |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual section label |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual section label |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual external resource note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**.
   - Node type: `Schedule Trigger`
   - Configure it to run every week.
   - Keep timezone considerations in mind based on your n8n instance settings.
   - Rename it to **Weekly Trigger**.

3. **Add a shipment source node**.
   - In this workflow, use a `Data Table` node.
   - Rename it to **Collect Shipments from TMS & WMS**.
   - Set:
     - Operation: `Get`
     - Return All: enabled
     - Data Table ID: select your shipment table
   - Connect **Weekly Trigger → Collect Shipments from TMS & WMS**.

4. **Ensure your source data contains these fields** for each shipment row:
   - `Order_Time`
   - `Delivery_OnTime`
   - `LT_Days`
   - `LT_Hours`
   - `Transmission_OnTime`
   - `Loading_OnTime`
   - `Airport_OnTime`
   - `Landing_OnTime`
   - Optional but useful: any shipment identifiers for future extensions

5. **Add a Code node** for weekly aggregation.
   - Rename it to **Aggregate by Week**.
   - Paste the aggregation logic that:
     - loads all shipment rows
     - groups by Monday-based week
     - computes KPI metrics
     - returns one item per week
   - Connect **Collect Shipments from TMS & WMS → Aggregate by Week**.

6. **Add an OpenAI Chat Model node** for weekly analysis.
   - Node type: `OpenAI Chat Model` from LangChain integration
   - Rename it to **OpenAI Chat Model**
   - Configure:
     - Model: `gpt-4o-mini`
     - Max Tokens: `300`
     - Temperature: `0.3`
   - Attach valid **OpenAI API credentials**.

7. **Add an AI Agent node** for weekly analysis.
   - Rename it to **AI Agent Weekly Performance Summary**.
   - Set prompt mode to define text manually.
   - Add this logic in plain form:
     - include week name, week start/end, shipment counts, lead time, and checkpoint percentages
     - ask for a brief 3–4 sentence analysis
     - request overall assessment, weakest checkpoint, and one recommendation
   - Add a system message instructing the model to act as a supply chain OTIF analyst, be concise, actionable, and use British spelling.
   - Connect:
     - **Aggregate by Week → AI Agent Weekly Performance Summary** via main connection
     - **OpenAI Chat Model → AI Agent Weekly Performance Summary** via AI language model connection

8. **Add a Notion node** for weekly KPI storage.
   - Rename it to **Fill the report**.
   - Resource: `Database Page`
   - Operation: create
   - Select your Notion credentials.
   - Select the target database, equivalent to **Daily OTIF Summary**.
   - Configure page icon as emoji `🚚`.
   - Map these properties:
     - `Name` as title from the aggregated week name
     - `Airport On-Time %`
     - `Avg Lead Time (days)`
     - `Avg Lead Time (hrs)`
     - `Date` as a date range using week start and week end
     - `Delivery On-Time %`
     - `Landing On-Time %`
     - `Late Shipments`
     - `Loading On-Time %`
     - `On-Time Deliveries`
     - `Total Shipments`
     - `Transmission On-Time %`
     - `AI Analysis` rich text property if desired
   - Use expressions referencing **Aggregate by Week** values, not the AI output.
   - Connect **AI Agent Weekly Performance Summary → Fill the report**.

9. **Create the first Notion database schema** before testing.
   - Database name can be `Daily OTIF Summary`.
   - Required properties and types:
     - `Name` → Title
     - `Airport On-Time %` → Number
     - `Avg Lead Time (days)` → Number
     - `Avg Lead Time (hrs)` → Number
     - `Date` → Date
     - `Delivery On-Time %` → Number
     - `Landing On-Time %` → Number
     - `Late Shipments` → Number
     - `Loading On-Time %` → Number
     - `On-Time Deliveries` → Number
     - `Total Shipments` → Number
     - `Transmission On-Time %` → Number
     - `AI Analysis` → Rich text

10. **Add a second Notion node** for weekly AI commentary.
    - Rename it to **Create Weekly Performance Card**.
    - Resource: `Database Page`
    - Operation: create
    - Choose the second Notion database, equivalent to **AI Analysis**.
    - Set icon as emoji `📊`.
    - Map:
      - `Name` from aggregated week name
      - `Weekly Analysis` from AI agent output
      - `Type` as select value `Weekly Analysis`
      - `Updated` as date
    - Connect **AI Agent Weekly Performance Summary → Create Weekly Performance Card**.

11. **Create the second Notion database schema**.
    - Database name can be `AI Analysis`.
    - Required properties and types:
      - `Name` → Title
      - `Weekly Analysis` → Rich text
      - `Type` → Select
      - `Updated` → Date
    - Add select options including:
      - `Weekly Analysis`
      - `Global Summary`

12. **Add a second Code node** for the global prompt.
    - Rename it to **Prepare Global Summary Prompt with Indicators**.
    - Configure it to:
      - collect all weekly aggregated items
      - sort them by `weekStart`
      - build one text block summarising all weeks
      - return a single item with a field called `globalPrompt`
    - Connect **Aggregate by Week → Prepare Global Summary Prompt with Indicators**.

13. **Add a second OpenAI Chat Model node**.
    - Rename it to **OpenAI Chat Model Global**.
    - Configure:
      - Model: `gpt-4o-mini`
      - Max Tokens: `600`
      - Temperature: `0.3`
    - Use valid OpenAI credentials.

14. **Add a second AI Agent node** for the global summary.
    - Rename it to **AI Agent Global Performance Summary**.
    - Configure prompt mode as defined text.
    - Use the incoming `globalPrompt` as the base prompt.
    - Ask for:
      - overall trend
      - recurring bottleneck checkpoint
      - best and worst weeks with reasons
      - two recommendations
      - short outlook
    - Ask for bullet points only and no markdown headers.
    - Use a system message focused on management-level OTIF analysis with British spelling.
    - Connect:
      - **Prepare Global Summary Prompt with Indicators → AI Agent Global Performance Summary**
      - **OpenAI Chat Model Global → AI Agent Global Performance Summary** via AI language model connection

15. **Add a final Notion node** to update the management summary card.
    - Rename it to **Update Global Performance Summary**.
    - Resource: `Database Page`
    - Operation: `Update`
    - Provide a fixed `Page ID` for the existing summary row.
    - Map:
      - `Name` = `Overall Performance Summary`
      - `Weekly Analysis` = AI agent output
      - `Type` = `Global Summary`
      - `Updated` = current date
    - Connect **AI Agent Global Performance Summary → Update Global Performance Summary**.

16. **Create or identify the dedicated Notion page to update**.
    - It must already exist in the analysis database or compatible target database.
    - Copy its page ID into the node.
    - Ensure the page contains compatible properties:
      - `Name`
      - `Weekly Analysis`
      - `Type`
      - `Updated`

17. **Add credentials**.
    - OpenAI credentials:
      - assign to both OpenAI Chat Model nodes
    - Notion credentials:
      - assign to all three Notion nodes

18. **Test the workflow with representative shipment data**.
    - Confirm the source returns rows.
    - Confirm the aggregation outputs one item per week.
    - Confirm the weekly AI agent outputs a field named `output`.
    - Confirm both weekly Notion pages are created.
    - Confirm the global AI agent returns one summary.
    - Confirm the final Notion page is updated.

19. **Validate expression references carefully**.
    - This workflow uses references like `$('Aggregate by Week').item.json.Name`.
    - If you change branching structure or insert merge/filter nodes, item linking may need adjustment.

20. **Add resilience improvements if rebuilding for production**.
    - Add an IF node after aggregation to stop when `error = No shipments found`.
    - Add deduplication logic in Notion if you do not want duplicate week entries.
    - Optionally write AI weekly output back into the KPI database’s `AI Analysis` field.
    - Consider limiting the global summary to a rolling date window if many weeks accumulate.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Supply Chain OTIF Performance Scorecard for Retail Logistics. Automatically aggregates shipment data, computes weekly OTIF KPIs, generates AI-powered performance analysis, and pushes everything to a Notion dashboard. | Workflow purpose |
| Setup checklist: connect your data source to the shipment collection node; add OpenAI API keys to both model nodes; add Notion API credentials to all Notion nodes; verify Notion database IDs. | Operational setup |
| Customisation ideas: adjust AI prompts to change analysis style or language; modify aggregation code to add custom KPIs. | Workflow maintenance and extension |
| Video resource | https://www.youtube.com/watch?v=tOT8XhQ7eB8 |