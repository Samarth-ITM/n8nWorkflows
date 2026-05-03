Score DNS threats with VirusTotal, Abuse.ch, HashiCorp Vault and Gemini

https://n8nworkflows.xyz/workflows/score-dns-threats-with-virustotal--abuse-ch--hashicorp-vault-and-gemini-14127


# Score DNS threats with VirusTotal, Abuse.ch, HashiCorp Vault and Gemini

# 1. Workflow Overview

This workflow implements an automated threat reputation pipeline for DNS/IP observables. Every 15 minutes, it pulls one pending record from a MySQL view/table, enriches that record with data from VirusTotal, ThreatFox, and URLhaus, stores raw evidence in MongoDB, reduces each provider’s response into a normalized summary, merges the summaries, sends the combined evidence to a Gemini-powered AI Agent, writes the AI verdict back to MySQL, and sends an alert email only when the resulting score is greater than 5.

Its main use case is passive DNS / cyber threat intelligence triage for domains and IPs, with secure secret retrieval via HashiCorp Vault and dual persistence across MongoDB and MySQL.

## 1.1 Orchestration & Data Ingestion

The workflow starts on a timer, retrieves secrets from HashiCorp Vault, and fetches one row from `cyber_intelligence.v_pending_analysis`. This row provides the observable IP, FQDN, and DNS query ID used throughout the rest of the execution.

## 1.2 Multi-Source Threat Intelligence Enrichment

The selected observable is queried against three external intelligence providers:

- VirusTotal for IP reputation
- Abuse.ch ThreatFox for IOC lookup by FQDN
- Abuse.ch URLhaus for host lookup by FQDN

Each branch uses conditional logic to determine whether the provider returned actionable data or effectively a clean/no-data result.

## 1.3 Dual-Layer Persistence Strategy

When a provider returns relevant data, the workflow transforms the response and stores the raw payload in MongoDB. It then reduces that raw payload into a compact normalized JSON structure suitable for AI consumption.

When a provider does not return useful data, the workflow records a “clear scan” directly into MySQL instead of inserting raw evidence into MongoDB.

## 1.4 AI Synthesis & Cognitive Analysis

The three normalized provider summaries are merged into one object and passed into an AI Agent backed by Google Gemini. The agent applies a strict scoring rubric and returns a single JSON verdict containing score, maliciousness, label, English summary, Polish analyst note, and active provider list.

## 1.5 Incident Reporting & Remediation

The AI output is parsed, matched back to provider MongoDB IDs, and persisted into relational tables in MySQL. If the score exceeds 5, the workflow retrieves email credentials from Vault and sends an HTML alert email with the analysis and evidence sources.

---

# 2. Block-by-Block Analysis

## 2.1 Orchestration & Data Ingestion

### Overview

This block triggers the workflow on a schedule, retrieves database-related secrets from HashiCorp Vault, and fetches one pending analysis record from MySQL. It establishes the working context for all downstream enrichment and persistence steps.

### Nodes Involved

- Schedule Trigger
- MySQL Pass
- Mongo DB Pass
- Select rows from a table

### Node Details

#### Schedule Trigger

- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry point that runs the workflow periodically.
- **Configuration choices:** Configured to run every 15 minutes.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **MySQL Pass**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:** Missed executions if n8n instance is down; timezone considerations depending on server settings.
- **Sub-workflow reference:** None.

#### MySQL Pass

- **Type and role:** `n8n-nodes-hashi-vault.hashiCorpVault`; retrieves MySQL secret material from Vault.
- **Configuration choices:** Secret path is `cyber-sentinel/credentials/mysql/app_manager`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Schedule Trigger**; output to **Mongo DB Pass**.
- **Version-specific requirements:** HashiCorp Vault node version `1`.
- **Edge cases / failures:** Vault authentication failure, missing secret path, insufficient Vault policy permissions.
- **Sub-workflow reference:** None.

#### Mongo DB Pass

- **Type and role:** `n8n-nodes-hashi-vault.hashiCorpVault`; retrieves MongoDB secret material from Vault.
- **Configuration choices:** Secret path is `cyber-sentinel/credentials/mongodb/admin`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **MySQL Pass**; output to **Select rows from a table**.
- **Version-specific requirements:** Version `1`.
- **Edge cases / failures:** Same Vault-related risks as above; note that the retrieved values are not explicitly mapped in this JSON to downstream node parameters, so this node may exist for secret-validation or external credential setup rather than runtime expression binding.
- **Sub-workflow reference:** None.

#### Select rows from a table

- **Type and role:** `n8n-nodes-base.mySql`; selects one pending analysis item from MySQL.
- **Configuration choices:** Operation `select`; table `cyber_intelligence.v_pending_analysis`; `limit: 1`.
- **Key expressions or variables used:** Downstream nodes reference:
  - `$('Select rows from a table').item.json.observable_ip`
  - `$('Select rows from a table').item.json.fqdn`
  - `$('Select rows from a table').item.json.dns_query_id`
- **Input and output connections:** Input from **Mongo DB Pass**; outputs to **Virus total API TOKEN** and **Abuse API TOKEN**.
- **Version-specific requirements:** MySQL node version `2.5`.
- **Edge cases / failures:** Empty result set; DB auth errors; connectivity issues; schema/view changes; if no row is returned, all downstream expressions that assume `.item.json` exists can fail.
- **Sub-workflow reference:** None.

---

## 2.2 Multi-Source Threat Intelligence Enrichment

### Overview

This block fans out into three external reputation/intelligence checks. Each provider branch uses its own retrieval and validation logic before either persisting raw evidence or logging a clean/no-data result.

### Nodes Involved

- Virus total API TOKEN
- VirusTotal IP Scan
- If 'malicious' or 'suspicious'
- Abuse API TOKEN
- Abuse.CH_ThreatFox request
- If query_status = ok - ThretFox
- Abuse.CH_URLHaus
- If query_status = ok - UrlHause

### Node Details

#### Virus total API TOKEN

- **Type and role:** HashiCorp Vault node; retrieves VirusTotal API secret.
- **Configuration choices:** Secret path `cyber-sentinel/api-keys/virustotal`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Select rows from a table**; output to **VirusTotal IP Scan**.
- **Version-specific requirements:** Version `1`.
- **Edge cases / failures:** Missing/expired Vault auth; wrong secret path.
- **Sub-workflow reference:** None.

#### VirusTotal IP Scan

- **Type and role:** `n8n-nodes-base.httpRequest`; performs GET request to VirusTotal IP endpoint.
- **Configuration choices:**
  - Method: `GET`
  - URL: `https://www.virustotal.com/api/v3/ip_addresses/{{ observable_ip }}`
  - Authentication: predefined credential type `virusTotalApi`
- **Key expressions or variables used:** `$('Select rows from a table').item.json.observable_ip`
- **Input and output connections:** Input from **Virus total API TOKEN**; output to **If 'malicious' or 'suspicious'**.
- **Version-specific requirements:** HTTP Request version `4.4`; extends `virusTotalApi`.
- **Edge cases / failures:** Invalid IP, API quota exhaustion, 401/403 auth errors, 429 rate limit, provider downtime. If VirusTotal returns an error object instead of the expected `data.attributes`, downstream expressions may fail.
- **Sub-workflow reference:** None.

#### If 'malicious' or 'suspicious'

- **Type and role:** `n8n-nodes-base.if`; evaluates whether VirusTotal produced meaningful malicious/suspicious signals.
- **Configuration choices:** OR condition:
  - `last_analysis_stats.malicious > 1`
  - `last_analysis_stats.suspicious > 1`
- **Key expressions or variables used:**
  - `$json.data.attributes.last_analysis_stats.malicious`
  - `$json.data.attributes.last_analysis_stats.suspicious`
- **Input and output connections:**
  - Input from **VirusTotal IP Scan**
  - True output to **Edit Json for Mongo - VirusTotal**
  - False output to **Insert a clear scan - VirusTotal**
- **Version-specific requirements:** IF node version `2.3`, condition options version `3`.
- **Edge cases / failures:** If `data.attributes.last_analysis_stats` is missing, expression evaluation can break. This node also treats low-count detections as effectively clean for persistence, which may suppress weak-signal evidence.
- **Sub-workflow reference:** None.

#### Abuse API TOKEN

- **Type and role:** HashiCorp Vault node; retrieves Abuse.ch API-related secret.
- **Configuration choices:** Secret path `cyber-sentinel/api-keys/abuse/api-key`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Select rows from a table**; outputs to **Abuse.CH_ThreatFox request** and **Abuse.CH_URLHaus**.
- **Version-specific requirements:** Version `1`.
- **Edge cases / failures:** Vault auth/policy/secret-path issues.
- **Sub-workflow reference:** None.

#### Abuse.CH_ThreatFox request

- **Type and role:** HTTP Request node; posts IOC search request to ThreatFox.
- **Configuration choices:**
  - URL: `https://threatfox-api.abuse.ch/api/v1/`
  - Method: `POST`
  - Body parameters:
    - `query = search_ioc`
    - `search_term = {{ fqdn }}`
    - `exact_match = true`
  - Authentication: generic credential type using HTTP header auth
- **Key expressions or variables used:** `$('Select rows from a table').item.json.fqdn`
- **Input and output connections:** Input from **Abuse API TOKEN**; output to **If query_status = ok - ThretFox**.
- **Version-specific requirements:** HTTP Request version `4.4`.
- **Edge cases / failures:** HTTP header auth misconfiguration; API schema changes; false negatives if searching FQDN where provider expects a different IOC format.
- **Sub-workflow reference:** None.

#### If query_status = ok - ThretFox

- **Type and role:** IF node; checks whether ThreatFox returned a successful query status.
- **Configuration choices:** Condition `query_status == "ok"`.
- **Key expressions or variables used:** `{{$json.query_status}}`
- **Input and output connections:**
  - Input from **Abuse.CH_ThreatFox request**
  - True output to **Edit Json for Mongo - ThreatFox**
  - False output to **Insert a clear scan - ThreatFox**
- **Version-specific requirements:** IF node version `2.3`.
- **Edge cases / failures:** Successful HTTP response with unexpected payload shape; note the node name contains a typo (`ThretFox`), which matters if referenced manually later.
- **Sub-workflow reference:** None.

#### Abuse.CH_URLHaus

- **Type and role:** HTTP Request node; queries URLhaus host endpoint by FQDN.
- **Configuration choices:**
  - URL: `https://urlhaus-api.abuse.ch/v1/host/`
  - Method: `POST`
  - Content type: form-urlencoded
  - Body parameter: `host = {{ fqdn }}`
  - Authentication: generic credential type using HTTP header auth
- **Key expressions or variables used:** `$('Select rows from a table').item.json.fqdn`
- **Input and output connections:** Input from **Abuse API TOKEN**; output to **If query_status = ok - UrlHause**.
- **Version-specific requirements:** HTTP Request version `4.4`.
- **Edge cases / failures:** Authentication or rate limit errors; malformed host value; provider may return empty but valid results.
- **Sub-workflow reference:** None.

#### If query_status = ok - UrlHause

- **Type and role:** IF node; validates URLhaus response status.
- **Configuration choices:** Condition `query_status == "ok"`.
- **Key expressions or variables used:** `{{$json.query_status}}`
- **Input and output connections:**
  - Input from **Abuse.CH_URLHaus**
  - True output to **Edit Json for Mongo - URLHaus**
  - False output to **Insert a clear scan - URLHaus**
- **Version-specific requirements:** IF node version `2.3`.
- **Edge cases / failures:** Node name includes misspelling (`UrlHause`), so any future expressions must match it exactly if referenced. A response can be `ok` but contain no URLs, which is later handled in the data reduction block.
- **Sub-workflow reference:** None.

---

## 2.3 Dual-Layer Persistence Strategy

### Overview

This block handles evidence persistence. Raw provider results judged relevant are transformed into MongoDB documents and then normalized into compact summaries; no-data or clean outcomes are recorded directly into MySQL as low-score records associated with their source provider.

### Nodes Involved

- Edit Json for Mongo - URLHaus
- Insert doc URLHaus
- Data reduction and aggregation - Urlhaus
- Insert a clear scan - URLHaus
- Edit Json for Mongo - ThreatFox
- Insert doc ThreatFox
- Data reduction and aggregation - ThreatFox
- Insert a clear scan - ThreatFox
- Edit Json for Mongo - VirusTotal
- Insert doc VirusTotal
- Data reduction and aggregation - VirusTotal
- Insert a clear scan - VirusTotal

### Node Details

#### Edit Json for Mongo - URLHaus

- **Type and role:** Code node; builds MongoDB document for URLhaus raw storage.
- **Configuration choices:** Returns object with:
  - `resource`: selected row’s `observable_ip`
  - `type`: `FQDN`
  - `source_provider`: `Abuse_URLhaus`
  - `scan_date`: current ISO timestamp
  - `raw_data`: full JSON from **Abuse.CH_URLHaus**
- **Key expressions or variables used:**
  - `$('Select rows from a table').first().json.observable_ip`
  - `$('Abuse.CH_URLHaus').item.json`
- **Input and output connections:** Input from **If query_status = ok - UrlHause** true branch; output to **Insert doc URLHaus**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** Uses `observable_ip` while `type` is `FQDN`, which is semantically inconsistent; if source node has no item, expressions fail.
- **Sub-workflow reference:** None.

#### Insert doc URLHaus

- **Type and role:** MongoDB node; stores URLhaus raw document.
- **Configuration choices:**
  - Operation: `insert`
  - Collection: `threat_data_raw`
  - Fields: `resource,type,source_provider,scan_date,raw_data`
- **Key expressions or variables used:** Uses incoming JSON fields.
- **Input and output connections:** Input from **Edit Json for Mongo - URLHaus**; output to **Data reduction and aggregation - Urlhaus**.
- **Version-specific requirements:** MongoDB node version `1.2`.
- **Edge cases / failures:** Mongo auth errors, collection permissions, oversized documents, schema/index constraints.
- **Sub-workflow reference:** None.

#### Data reduction and aggregation - Urlhaus

- **Type and role:** Code node; reduces raw URLhaus data into normalized AI-ready structure.
- **Configuration choices:**
  - Reads `raw_data.urls`
  - Counts online URLs
  - Aggregates tags and highlights critical keywords such as `c2`, `rat`, `ransomware`, `infostealer`
  - Emits:
    - `urlhaus.no_data`
    - `urlhaus.urlhaus_report`
    - `urlhaus.is_active_threat`
    - `urlhaus.urlhaus_reference`
    - `urlhaus.mongo_id`
- **Key expressions or variables used:** `$input.item.json`, `item.id`
- **Input and output connections:** Input from **Insert doc URLHaus**; output to **Merge** input 0.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** If provider payload structure changes, report generation may silently degrade. `mongo_id` assumes inserted Mongo item exposes `id` in output.
- **Sub-workflow reference:** None.

#### Insert a clear scan - URLHaus

- **Type and role:** MySQL node; records a clean/no-data URLhaus scan into relational tables.
- **Configuration choices:** Executes multi-statement SQL:
  1. Insert low-score result into `ai_analysis_results`
  2. Insert event into `threat_indicators`
  3. Insert source detail into `threat_indicator_details` with `mongo_ref_id = 'NO_DATA'`
- **Key expressions or variables used:** `$('Select rows from a table').item.json.dns_query_id`
- **Input and output connections:** Input from **If query_status = ok - UrlHause** false branch.
- **Version-specific requirements:** MySQL node version `2.5`; database connection must allow multi-statements.
- **Edge cases / failures:** Multi-statement execution may be disabled in some MySQL configurations; if relational dictionary values are missing (`dic_indicator_types`, `dic_source_providers`), inserts fail.
- **Sub-workflow reference:** None.

#### Edit Json for Mongo - ThreatFox

- **Type and role:** Code node; creates MongoDB storage object for ThreatFox.
- **Configuration choices:** Returns:
  - `resource`: `observable_ip`
  - `type`: `FQDN`
  - `source_provider`: `Abuse_ThreatFox`
  - `scan_date`: current ISO timestamp
  - `raw_data`: full ThreatFox response
- **Key expressions or variables used:**
  - `$('Select rows from a table').first().json.observable_ip`
  - `$('Abuse.CH_ThreatFox request').item.json`
- **Input and output connections:** Input from **If query_status = ok - ThretFox** true branch; output to **Insert doc ThreatFox**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** Same semantic mismatch as above: `resource` uses IP while lookup used FQDN and type is `FQDN`.
- **Sub-workflow reference:** None.

#### Insert doc ThreatFox

- **Type and role:** MongoDB node; stores ThreatFox raw document.
- **Configuration choices:** Insert into `threat_data_raw`.
- **Key expressions or variables used:** Incoming fields.
- **Input and output connections:** Input from **Edit Json for Mongo - ThreatFox**; output to **Data reduction and aggregation - ThreatFox**.
- **Version-specific requirements:** Version `1.2`.
- **Edge cases / failures:** Standard MongoDB insert issues.
- **Sub-workflow reference:** None.

#### Data reduction and aggregation - ThreatFox

- **Type and role:** Code node; summarizes ThreatFox findings into normalized format.
- **Configuration choices:**
  - Extracts malware families and threat types
  - Computes average confidence
  - Aggregates tags
  - Includes external reference
  - Emits:
    - `threatfox.no_data`
    - `threatfox.threatfox_report`
    - `threatfox.threatfox_active`
    - `threatfox.threatfox_max_confidence`
    - `threatfox.mongo_id`
- **Key expressions or variables used:** `$input.item.json`, `item.id`
- **Input and output connections:** Input from **Insert doc ThreatFox**; output to **Merge** input 1.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** `confidence_level` may be non-numeric or absent; status logic uses `raw.query_status !== "ok"` and empty list handling, which is good but still assumes the top-level structure is stable.
- **Sub-workflow reference:** None.

#### Insert a clear scan - ThreatFox

- **Type and role:** MySQL node; stores a clean/no-data ThreatFox event.
- **Configuration choices:** Multi-statement SQL similar to URLhaus branch; uses source provider `Abuse_ThreatFox`.
- **Key expressions or variables used:**
  - `$('Select rows from a table').item.json.dns_query_id`
  - `{{$json.mongo_id || "NO_DATA"}}`
- **Input and output connections:** Input from **If query_status = ok - ThretFox** false branch.
- **Version-specific requirements:** MySQL `2.5`.
- **Edge cases / failures:** Same dictionary and multi-statement concerns; if false branch item lacks `mongo_id`, falls back safely to `NO_DATA`.
- **Sub-workflow reference:** None.

#### Edit Json for Mongo - VirusTotal

- **Type and role:** Code node; creates MongoDB document for VirusTotal evidence.
- **Configuration choices:** Returns:
  - `resource`: `$('If 'malicious' or 'suspicious'').item.json.data.id`
  - `type`: `IP`
  - `source_provider`: `VirusTotal`
  - `scan_date`: current ISO timestamp
  - `raw_data`: `$('VirusTotal IP Scan').item.json.data`
- **Key expressions or variables used:** References VirusTotal and IF node outputs.
- **Input and output connections:** Input from **If 'malicious' or 'suspicious'** true branch; output to **Insert doc VirusTotal**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** If VirusTotal response lacks `data.id`, this node breaks. Also depends on the true branch having data with the expected shape.
- **Sub-workflow reference:** None.

#### Insert doc VirusTotal

- **Type and role:** MongoDB node; stores VirusTotal raw document.
- **Configuration choices:** Insert into `threat_data_raw`.
- **Key expressions or variables used:** Incoming fields.
- **Input and output connections:** Input from **Edit Json for Mongo - VirusTotal**; output to **Data reduction and aggregation - VirusTotal**.
- **Version-specific requirements:** Version `1.2`.
- **Edge cases / failures:** Standard Mongo insert concerns.
- **Sub-workflow reference:** None.

#### Data reduction and aggregation - VirusTotal

- **Type and role:** Code node; reduces VirusTotal raw record into normalized summary.
- **Configuration choices:**
  - Extracts `last_analysis_stats`
  - Builds flagged engine list from `last_analysis_results`
  - Extracts ownership/network/RDAP range
  - Flags large trusted providers using owner keywords: `github`, `google`, `microsoft`, `amazon`, `cloudflare`
  - Emits:
    - `virustotal.no_data`
    - `virustotal.vt_report`
    - `virustotal.vt_stats`
    - `virustotal.vt_owner`
    - `virustotal.vt_is_big_player`
    - `virustotal.vt_malicious_count`
    - `virustotal.vt_scan_date`
    - `virustotal.mongo_id`
- **Key expressions or variables used:** `$input.item.json`, `item.raw_data.attributes`
- **Input and output connections:** Input from **Insert doc VirusTotal**; output to **Merge** input 2.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** If ownership field is absent, it falls back to `"Unknown Owner"`; if `raw_data.attributes` is missing, node returns a no-data structure.
- **Sub-workflow reference:** None.

#### Insert a clear scan - VirusTotal

- **Type and role:** MySQL node; records a neutral VirusTotal scan when the VT threshold is not met.
- **Configuration choices:** Multi-statement SQL inserting score `1`, threat indicator, and source detail for `VirusTotal`.
- **Key expressions or variables used:**
  - `$('Select rows from a table').item.json.dns_query_id`
  - `{{$json.mongo_id || "NO_DATA"}}`
- **Input and output connections:** Input from **If 'malicious' or 'suspicious'** false branch.
- **Version-specific requirements:** MySQL `2.5`.
- **Edge cases / failures:** This branch treats low detections as clean, which may create false negatives in the relational record layer even though the AI branch will receive only data from the positive path.
- **Sub-workflow reference:** None.

---

## 2.4 AI Synthesis & Cognitive Analysis

### Overview

This block merges the normalized provider outputs, passes them to a Gemini-backed AI Agent with a strict cyber scoring rubric, and parses the returned JSON into a database-friendly structure enriched with provider-to-Mongo references.

### Nodes Involved

- Merge
- Code for Merge
- Gemini AI Studio
- Google Gemini Chat Model
- AI Agent
- Parse AI Agent output

### Node Details

#### Merge

- **Type and role:** `n8n-nodes-base.merge`; collects three provider summaries.
- **Configuration choices:** `numberInputs: 3`.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input 0 from **Data reduction and aggregation - Urlhaus**
  - Input 1 from **Data reduction and aggregation - ThreatFox**
  - Input 2 from **Data reduction and aggregation - VirusTotal**
  - Output to **Code for Merge**
- **Version-specific requirements:** Merge version `3.2`.
- **Edge cases / failures:** If one or more provider summary nodes never execute because the branch ended in “clear scan” SQL instead of normalized reduction, the merge behavior depends on n8n execution semantics and may stall or produce incomplete input. This is a key architectural risk in the current design.
- **Sub-workflow reference:** None.

#### Code for Merge

- **Type and role:** Code node; combines all merge inputs into one JSON object.
- **Configuration choices:** Iterates over `$input.all()` and `Object.assign`s each item JSON into `combinedData`.
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:** Input from **Merge**; output to **Gemini AI Studio**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:** Later objects can overwrite earlier keys if duplicate top-level properties exist; assumes each input contains unique provider keys.
- **Sub-workflow reference:** None.

#### Gemini AI Studio

- **Type and role:** HashiCorp Vault node; retrieves Gemini API secret from Vault.
- **Configuration choices:** Secret path `cyber-sentinel/api-keys/gemini/home-network-guardian`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Code for Merge**; output to **AI Agent**.
- **Version-specific requirements:** Version `1`.
- **Edge cases / failures:** Vault access issues; note that a dedicated Gemini credential is also directly configured on the model node, so this node may be used for secret orchestration but not explicit runtime parameter injection.
- **Sub-workflow reference:** None.

#### Google Gemini Chat Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; language model attached to the AI Agent.
- **Configuration choices:** Default options; credential type `googlePalmApi`.
- **Key expressions or variables used:** None directly.
- **Input and output connections:** Connected to **AI Agent** through `ai_languageModel`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** Invalid API key, quota exhaustion, model/API deprecations, non-JSON output despite instruction.
- **Sub-workflow reference:** None.

#### AI Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; synthesizes multi-source threat intelligence into one structured verdict.
- **Configuration choices:**
  - `promptType: define`
  - `hasOutputParser: true`
  - `executeOnce: true`
  - Very detailed prompt with:
    - score rubric 1–10
    - source-specific interpretation rules
    - no-data logic
    - trusted-provider exception
    - strict JSON output format
    - SQL escaping rules
- **Key expressions or variables used:** `{{ JSON.stringify($('Code for Merge').item.json) }}`
- **Input and output connections:** Main input from **Gemini AI Studio**; language model input from **Google Gemini Chat Model**; main output to **Parse AI Agent output**.
- **Version-specific requirements:** Agent version `3.1`.
- **Edge cases / failures:** Model may still emit markdown fences or malformed JSON; prompt asks for SQL-safe escaping but does not guarantee it; long provider reports may increase latency/cost.
- **Sub-workflow reference:** None.

#### Parse AI Agent output

- **Type and role:** Code node; sanitizes and parses AI JSON output and reconstructs provider references.
- **Configuration choices:**
  - Removes ```json fences
  - Replaces line breaks with spaces
  - Parses JSON
  - Builds `provider_details` array by scanning outputs of:
    - `Data reduction and aggregation - VirusTotal`
    - `Data reduction and aggregation - ThreatFox`
    - `Data reduction and aggregation - Urlhaus`
- **Key expressions or variables used:**
  - `$json.output`
  - `$(nodeName).all()`
  - `data.active_providers.includes(...)`
- **Input and output connections:** Input from **AI Agent**; output to **Insert AI verdict**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases / failures:**
  - If AI output is invalid JSON, node returns error object instead of expected fields.
  - If provider aggregation nodes did not run, Mongo ID lookup may fail and `provider_details` may be incomplete.
  - Downstream SQL expects `score`, `verdict_en`, and `analysis_pl`; parsing failure could break inserts.
- **Sub-workflow reference:** None.

---

## 2.5 Incident Reporting & Remediation

### Overview

This block persists the AI verdict to MySQL, filters for actionable risk, retrieves email credentials, and sends an HTML alert. It is the final notification and incident registration stage.

### Nodes Involved

- Insert AI verdict
- Filter
- Get Email Pass
- Send an Email

### Node Details

#### Insert AI verdict

- **Type and role:** MySQL node; writes AI verdict and provider evidence mappings into relational storage.
- **Configuration choices:** Multi-statement SQL that:
  1. Inserts verdict into `ai_analysis_results`
  2. Inserts event into `threat_indicators`
  3. Inserts one row per provider into `threat_indicator_details`
  4. Returns `indicator_id`
- **Key expressions or variables used:**
  - `{{$json.score}}`
  - `{{$json.verdict_en}}`
  - `{{$json.analysis_pl}}`
  - `$('Select rows from a table').item.json.dns_query_id`
  - `{{$json.provider_details.map(...)}}`
- **Input and output connections:** Input from **Parse AI Agent output**; output to **Filter**.
- **Version-specific requirements:** MySQL `2.5`; `alwaysOutputData: true`.
- **Edge cases / failures:**
  - SQL injection / quoting issues if model output is not correctly escaped despite prompt instructions.
  - Requires MySQL driver support for multi-statements.
  - Provider names must exactly match values in `dic_source_providers`.
- **Sub-workflow reference:** None.

#### Filter

- **Type and role:** `n8n-nodes-base.filter`; routes only medium/high-risk results to alerting.
- **Configuration choices:** Condition `score > 5`.
- **Key expressions or variables used:** `$('Parse AI Agent output').item.json.score`
- **Input and output connections:** Input from **Insert AI verdict**; output to **Get Email Pass**.
- **Version-specific requirements:** Filter version `2.3`.
- **Edge cases / failures:** If parse node returned error object, `score` may be undefined and filtering may fail or skip alerting.
- **Sub-workflow reference:** None.

#### Get Email Pass

- **Type and role:** HashiCorp Vault node; retrieves email credentials/secret material.
- **Configuration choices:** Secret path `cyber-sentinel/credentials/gmail/l94524506`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Filter**; output to **Send an Email**.
- **Version-specific requirements:** Version `1`.
- **Edge cases / failures:** Vault secret issues; again, SMTP credentials are also directly configured in the email node, so this may be a secret-orchestration placeholder.
- **Sub-workflow reference:** None.

#### Send an Email

- **Type and role:** `n8n-nodes-base.emailSend`; sends formatted HTML incident alert.
- **Configuration choices:**
  - Subject includes current Polish locale timestamp
  - Dark-themed HTML card
  - Displays score, IP, FQDN, active sources, Polish analysis, technical verdict
  - Includes RDAP link to ARIN for the IP
  - Uses SMTP credentials
- **Key expressions or variables used:**
  - `$node["Parse AI Agent output"].json.score`
  - `$('Select rows from a table').item.json.observable_ip`
  - `$('Select rows from a table').item.json.fqdn`
  - `$node["Parse AI Agent output"].json.provider_details`
  - `$node["Parse AI Agent output"].json.analysis_pl`
  - `$node["Parse AI Agent output"].json.verdict_en`
  - `new Date().toLocaleString('pl-PL')`
- **Input and output connections:** Input from **Get Email Pass**; no downstream nodes.
- **Version-specific requirements:** Email Send version `2.1`.
- **Edge cases / failures:** SMTP auth issues; HTML rendering varies by client; uses hard-coded `fromEmail` and `toEmail` placeholders (`user@example.com`) that must be replaced before production use.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Timed workflow entry point every 15 minutes |  | MySQL Pass | ## Orchestration & Data Ingestion |
| MySQL Pass | n8n-nodes-hashi-vault.hashiCorpVault | Retrieve MySQL secret from HashiCorp Vault | Schedule Trigger | Mongo DB Pass | ## Orchestration & Data Ingestion |
| Mongo DB Pass | n8n-nodes-hashi-vault.hashiCorpVault | Retrieve MongoDB secret from HashiCorp Vault | MySQL Pass | Select rows from a table | ## Orchestration & Data Ingestion |
| Select rows from a table | n8n-nodes-base.mySql | Fetch one pending observable from MySQL | Mongo DB Pass | Virus total API TOKEN; Abuse API TOKEN | ## Orchestration & Data Ingestion |
| Virus total API TOKEN | n8n-nodes-hashi-vault.hashiCorpVault | Retrieve VirusTotal secret | Select rows from a table | VirusTotal IP Scan | ## Multi-Source Threat Intelligence Enrichment |
| VirusTotal IP Scan | n8n-nodes-base.httpRequest | Query VirusTotal IP reputation | Virus total API TOKEN | If 'malicious' or 'suspicious' | ## Multi-Source Threat Intelligence Enrichment |
| If 'malicious' or 'suspicious' | n8n-nodes-base.if | Branch on VT malicious/suspicious thresholds | VirusTotal IP Scan | Edit Json for Mongo - VirusTotal; Insert a clear scan - VirusTotal | ## Multi-Source Threat Intelligence Enrichment |
| Abuse API TOKEN | n8n-nodes-hashi-vault.hashiCorpVault | Retrieve Abuse.ch secret | Select rows from a table | Abuse.CH_ThreatFox request; Abuse.CH_URLHaus | ## Multi-Source Threat Intelligence Enrichment |
| Abuse.CH_ThreatFox request | n8n-nodes-base.httpRequest | Query ThreatFox for IOC matches on FQDN | Abuse API TOKEN | If query_status = ok - ThretFox | ## Multi-Source Threat Intelligence Enrichment |
| If query_status = ok - ThretFox | n8n-nodes-base.if | Branch on ThreatFox query status | Abuse.CH_ThreatFox request | Edit Json for Mongo - ThreatFox; Insert a clear scan - ThreatFox | ## Multi-Source Threat Intelligence Enrichment |
| Abuse.CH_URLHaus | n8n-nodes-base.httpRequest | Query URLhaus host data for FQDN | Abuse API TOKEN | If query_status = ok - UrlHause | ## Multi-Source Threat Intelligence Enrichment |
| If query_status = ok - UrlHause | n8n-nodes-base.if | Branch on URLhaus query status | Abuse.CH_URLHaus | Edit Json for Mongo - URLHaus; Insert a clear scan - URLHaus | ## Multi-Source Threat Intelligence Enrichment |
| Edit Json for Mongo - URLHaus | n8n-nodes-base.code | Build MongoDB raw evidence document for URLhaus | If query_status = ok - UrlHause | Insert doc URLHaus | ## Dual-Layer Persistence Strategy |
| Insert doc URLHaus | n8n-nodes-base.mongoDb | Store URLhaus raw response in MongoDB | Edit Json for Mongo - URLHaus | Data reduction and aggregation - Urlhaus | ## Dual-Layer Persistence Strategy |
| Data reduction and aggregation - Urlhaus | n8n-nodes-base.code | Normalize URLhaus raw data for AI scoring | Insert doc URLHaus | Merge | ## Dual-Layer Persistence Strategy |
| Insert a clear scan - URLHaus | n8n-nodes-base.mySql | Persist no-data/clean URLhaus result to MySQL | If query_status = ok - UrlHause |  | ## Dual-Layer Persistence Strategy |
| Edit Json for Mongo - ThreatFox | n8n-nodes-base.code | Build MongoDB raw evidence document for ThreatFox | If query_status = ok - ThretFox | Insert doc ThreatFox | ## Dual-Layer Persistence Strategy |
| Insert doc ThreatFox | n8n-nodes-base.mongoDb | Store ThreatFox raw response in MongoDB | Edit Json for Mongo - ThreatFox | Data reduction and aggregation - ThreatFox | ## Dual-Layer Persistence Strategy |
| Data reduction and aggregation - ThreatFox | n8n-nodes-base.code | Normalize ThreatFox raw data for AI scoring | Insert doc ThreatFox | Merge | ## Dual-Layer Persistence Strategy |
| Insert a clear scan - ThreatFox | n8n-nodes-base.mySql | Persist no-data/clean ThreatFox result to MySQL | If query_status = ok - ThretFox |  | ## Dual-Layer Persistence Strategy |
| Edit Json for Mongo - VirusTotal | n8n-nodes-base.code | Build MongoDB raw evidence document for VirusTotal | If 'malicious' or 'suspicious' | Insert doc VirusTotal | ## Dual-Layer Persistence Strategy |
| Insert doc VirusTotal | n8n-nodes-base.mongoDb | Store VirusTotal raw response in MongoDB | Edit Json for Mongo - VirusTotal | Data reduction and aggregation - VirusTotal | ## Dual-Layer Persistence Strategy |
| Data reduction and aggregation - VirusTotal | n8n-nodes-base.code | Normalize VirusTotal raw data for AI scoring | Insert doc VirusTotal | Merge | ## Dual-Layer Persistence Strategy |
| Insert a clear scan - VirusTotal | n8n-nodes-base.mySql | Persist low-signal/clean VirusTotal result to MySQL | If 'malicious' or 'suspicious' |  | ## Dual-Layer Persistence Strategy |
| Merge | n8n-nodes-base.merge | Collect normalized outputs from three providers | Data reduction and aggregation - Urlhaus; Data reduction and aggregation - ThreatFox; Data reduction and aggregation - VirusTotal | Code for Merge | ## AI Synthesis & Cognitive Analysis |
| Code for Merge | n8n-nodes-base.code | Merge provider JSON objects into one payload | Merge | Gemini AI Studio | ## AI Synthesis & Cognitive Analysis |
| Gemini AI Studio | n8n-nodes-hashi-vault.hashiCorpVault | Retrieve Gemini secret from Vault | Code for Merge | AI Agent | ## AI Synthesis & Cognitive Analysis |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for the AI Agent |  | AI Agent | ## AI Synthesis & Cognitive Analysis |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate final threat score and analysis JSON | Gemini AI Studio; Google Gemini Chat Model | Parse AI Agent output | ## AI Synthesis & Cognitive Analysis |
| Parse AI Agent output | n8n-nodes-base.code | Parse AI JSON and map source providers to Mongo IDs | AI Agent | Insert AI verdict | ## Incident Reporting & Remediation |
| Insert AI verdict | n8n-nodes-base.mySql | Store AI verdict and source mappings in MySQL | Parse AI Agent output | Filter | ## Incident Reporting & Remediation |
| Filter | n8n-nodes-base.filter | Forward only alerts with score > 5 | Insert AI verdict | Get Email Pass | ## Incident Reporting & Remediation |
| Get Email Pass | n8n-nodes-hashi-vault.hashiCorpVault | Retrieve email-related secret from Vault | Filter | Send an Email | ## Incident Reporting & Remediation |
| Send an Email | n8n-nodes-base.emailSend | Send HTML incident alert via SMTP | Get Email Pass |  | ## Incident Reporting & Remediation |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for orchestration block |  |  | ## Orchestration & Data Ingestion |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for enrichment block |  |  | ## Multi-Source Threat Intelligence Enrichment |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for persistence block |  |  | ## Dual-Layer Persistence Strategy |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for AI block |  |  | ## AI Synthesis & Cognitive Analysis |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for reporting block |  |  | ## Incident Reporting & Remediation |
| Sticky Note5 | n8n-nodes-base.stickyNote | Project note and setup context |  |  | # 🛡️ Project: Cyber Sentinel<br>### **Purpose**<br>Autonomous **Cyber Threat Intelligence (CTI)** & **Passive DNS Monitoring**. This workflow orchestrates the analysis of DNS telemetry using AI to detect malicious patterns and trigger autonomous response playbooks.<br>---<br>### **🚀 Setup Instructions**<br>1. **Secret Management:** Ensure **HashiCorp Vault** is connected to handle API keys and credentials securely.<br>2. **AI Engine:** Configure the **Ollama/Gemini** node to enable bilingual threat assessment (**EN/PL**).<br>3. **Data Layer:** Verify the connection to the **PostgreSQL** database (Neural Lake) for indicator enrichment.<br>---<br>### **🔗 Resources**<br>Documentation: [https://lukaszfd.github.io/cyber-sentinel/](https://lukaszfd.github.io/cyber-sentinel/)<br>Author: Łukasz Dejko |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence.

## 4.1 Prepare credentials first

1. **Create a HashiCorp Vault credential**
   - Use the `HashiCorp Vault` credential type required by `n8n-nodes-hashi-vault.hashiCorpVault`.
   - Ensure it has read access to:
     - `cyber-sentinel/credentials/mysql/app_manager`
     - `cyber-sentinel/credentials/mongodb/admin`
     - `cyber-sentinel/api-keys/virustotal`
     - `cyber-sentinel/api-keys/abuse/api-key`
     - `cyber-sentinel/api-keys/gemini/home-network-guardian`
     - `cyber-sentinel/credentials/gmail/l94524506`

2. **Create a MySQL credential**
   - Configure access to the database containing:
     - `cyber_intelligence.v_pending_analysis`
     - `cyber_intelligence.ai_analysis_results`
     - `cyber_intelligence.threat_indicators`
     - `cyber_intelligence.threat_indicator_details`
     - `cyber_intelligence.dic_indicator_types`
     - `cyber_intelligence.dic_source_providers`
   - Ensure multi-statement SQL execution is supported.

3. **Create a MongoDB credential**
   - Configure connection to the database containing collection `threat_data_raw`.

4. **Create a VirusTotal credential**
   - Use the dedicated `virusTotalApi` credential type.

5. **Create an HTTP Header Auth credential for Abuse.ch**
   - Use it for both ThreatFox and URLhaus requests if required by your environment.
   - If the API does not actually require auth in your deployment, you may still keep the node configured as in the original workflow.

6. **Create a Google Gemini credential**
   - Credential type: `Google Gemini(PaLM) Api account` / `googlePalmApi`.

7. **Create an SMTP credential**
   - For the email node.
   - Replace placeholder sender/recipient addresses.

---

## 4.2 Build the orchestration layer

1. **Add `Schedule Trigger`**
   - Type: `Schedule Trigger`
   - Set interval to every `15 minutes`.

2. **Add `MySQL Pass`**
   - Type: `HashiCorp Vault`
   - Secret path: `cyber-sentinel/credentials/mysql/app_manager`

3. **Add `Mongo DB Pass`**
   - Type: `HashiCorp Vault`
   - Secret path: `cyber-sentinel/credentials/mongodb/admin`

4. **Add `Select rows from a table`**
   - Type: `MySQL`
   - Operation: `Select`
   - Table: `cyber_intelligence.v_pending_analysis`
   - Limit: `1`

5. **Connect**
   - `Schedule Trigger` → `MySQL Pass`
   - `MySQL Pass` → `Mongo DB Pass`
   - `Mongo DB Pass` → `Select rows from a table`

---

## 4.3 Build the VirusTotal branch

1. **Add `Virus total API TOKEN`**
   - Type: `HashiCorp Vault`
   - Secret path: `cyber-sentinel/api-keys/virustotal`

2. **Add `VirusTotal IP Scan`**
   - Type: `HTTP Request`
   - Method: `GET`
   - URL:
     ```text
     =https://www.virustotal.com/api/v3/ip_addresses/{{ $('Select rows from a table').item.json.observable_ip }}
     ```
   - Authentication: `Predefined Credential Type`
   - Credential type: `virusTotalApi`

3. **Add `If 'malicious' or 'suspicious'`**
   - Type: `IF`
   - Condition group combinator: `OR`
   - Condition 1:
     - Left value: `={{ $json.data.attributes.last_analysis_stats.malicious }}`
     - Operator: `>`
     - Right value: `1`
   - Condition 2:
     - Left value: `={{ $json.data.attributes.last_analysis_stats.suspicious }}`
     - Operator: `>`
     - Right value: `1`

4. **Add `Edit Json for Mongo - VirusTotal`**
   - Type: `Code`
   - JS:
     ```javascript
     return {
       resource: $('If \'malicious\' or \'suspicious\'').item.json.data.id, 
       type: "IP",
       source_provider: "VirusTotal", 
       scan_date: new Date().toISOString(),
       raw_data: $('VirusTotal IP Scan').item.json.data 
     };
     ```

5. **Add `Insert doc VirusTotal`**
   - Type: `MongoDB`
   - Operation: `Insert`
   - Collection: `threat_data_raw`
   - Fields:
     ```text
     =resource,type,source_provider,scan_date,raw_data
     ```

6. **Add `Data reduction and aggregation - VirusTotal`**
   - Type: `Code`
   - JS: use the exact normalization logic from the workflow JSON.

7. **Add `Insert a clear scan - VirusTotal`**
   - Type: `MySQL`
   - Operation: `Execute Query`
   - Paste the SQL from the original node.
   - Ensure source provider dictionary contains `VirusTotal`.

8. **Connect**
   - `Select rows from a table` → `Virus total API TOKEN`
   - `Virus total API TOKEN` → `VirusTotal IP Scan`
   - `VirusTotal IP Scan` → `If 'malicious' or 'suspicious'`
   - True → `Edit Json for Mongo - VirusTotal` → `Insert doc VirusTotal` → `Data reduction and aggregation - VirusTotal`
   - False → `Insert a clear scan - VirusTotal`

---

## 4.4 Build the ThreatFox branch

1. **Add `Abuse API TOKEN`**
   - Type: `HashiCorp Vault`
   - Secret path: `cyber-sentinel/api-keys/abuse/api-key`

2. **Add `Abuse.CH_ThreatFox request`**
   - Type: `HTTP Request`
   - Method: `POST`
   - URL: `https://threatfox-api.abuse.ch/api/v1/`
   - Authentication: `Generic Credential Type`
   - Generic auth type: `HTTP Header Auth`
   - Body parameters:
     - `query = search_ioc`
     - `search_term = {{ $('Select rows from a table').item.json.fqdn }}`
     - `exact_match = true`

3. **Add `If query_status = ok - ThretFox`**
   - Type: `IF`
   - Condition:
     - Left value: `={{ $json.query_status }}`
     - Operator: `equals`
     - Right value: `ok`

4. **Add `Edit Json for Mongo - ThreatFox`**
   - Type: `Code`
   - JS:
     ```javascript
     return {
       resource: $('Select rows from a table').first().json.observable_ip,
       type: 'FQDN',
       source_provider: 'Abuse_ThreatFox', 
       scan_date: new Date().toISOString(),
       raw_data: $('Abuse.CH_ThreatFox request').item.json
     };
     ```

5. **Add `Insert doc ThreatFox`**
   - Type: `MongoDB`
   - Operation: `Insert`
   - Collection: `threat_data_raw`
   - Fields: `=resource,type,source_provider,scan_date,raw_data`

6. **Add `Data reduction and aggregation - ThreatFox`**
   - Type: `Code`
   - JS: use the original summarization logic.

7. **Add `Insert a clear scan - ThreatFox`**
   - Type: `MySQL`
   - Operation: `Execute Query`
   - Paste the SQL from the original node.

8. **Connect**
   - `Select rows from a table` → `Abuse API TOKEN`
   - `Abuse API TOKEN` → `Abuse.CH_ThreatFox request`
   - `Abuse.CH_ThreatFox request` → `If query_status = ok - ThretFox`
   - True → `Edit Json for Mongo - ThreatFox` → `Insert doc ThreatFox` → `Data reduction and aggregation - ThreatFox`
   - False → `Insert a clear scan - ThreatFox`

---

## 4.5 Build the URLhaus branch

1. **Add `Abuse.CH_URLHaus`**
   - Type: `HTTP Request`
   - Method: `POST`
   - URL: `https://urlhaus-api.abuse.ch/v1/host/`
   - Authentication: `Generic Credential Type`
   - Generic auth type: `HTTP Header Auth`
   - Content type: `Form URL Encoded`
   - Body parameter:
     - `host = {{ $('Select rows from a table').item.json.fqdn }}`

2. **Add `If query_status = ok - UrlHause`**
   - Type: `IF`
   - Condition:
     - Left value: `={{ $json.query_status }}`
     - Operator: `equals`
     - Right value: `ok`

3. **Add `Edit Json for Mongo - URLHaus`**
   - Type: `Code`
   - JS:
     ```javascript
     return {
       resource: $('Select rows from a table').first().json.observable_ip,
       type: 'FQDN',
       source_provider: 'Abuse_URLhaus', 
       scan_date: new Date().toISOString(),
       raw_data: $('Abuse.CH_URLHaus').item.json
     };
     ```

4. **Add `Insert doc URLHaus`**
   - Type: `MongoDB`
   - Operation: `Insert`
   - Collection: `threat_data_raw`
   - Fields: `=resource,type,source_provider,scan_date,raw_data`

5. **Add `Data reduction and aggregation - Urlhaus`**
   - Type: `Code`
   - JS: use the original normalization logic.

6. **Add `Insert a clear scan - URLHaus`**
   - Type: `MySQL`
   - Operation: `Execute Query`
   - Paste the SQL from the original node.

7. **Connect**
   - `Abuse API TOKEN` → `Abuse.CH_URLHaus`
   - `Abuse.CH_URLHaus` → `If query_status = ok - UrlHause`
   - True → `Edit Json for Mongo - URLHaus` → `Insert doc URLHaus` → `Data reduction and aggregation - Urlhaus`
   - False → `Insert a clear scan - URLHaus`

---

## 4.6 Build the merge and AI layer

1. **Add `Merge`**
   - Type: `Merge`
   - Number of inputs: `3`

2. **Connect provider summaries**
   - `Data reduction and aggregation - Urlhaus` → `Merge` input 1
   - `Data reduction and aggregation - ThreatFox` → `Merge` input 2
   - `Data reduction and aggregation - VirusTotal` → `Merge` input 3

3. **Add `Code for Merge`**
   - Type: `Code`
   - JS:
     ```javascript
     let combinedData = {};

     for (const item of $input.all()) {
       Object.assign(combinedData, item.json);
     }

     return combinedData;
     ```

4. **Add `Gemini AI Studio`**
   - Type: `HashiCorp Vault`
   - Secret path: `cyber-sentinel/api-keys/gemini/home-network-guardian`

5. **Add `Google Gemini Chat Model`**
   - Type: `Google Gemini Chat Model`
   - Attach Google Gemini credentials

6. **Add `AI Agent`**
   - Type: `AI Agent`
   - Prompt type: `Define`
   - Execute once: enabled
   - Has output parser: enabled
   - Paste the full scoring prompt from the original node exactly, because it contains:
     - scoring rubric
     - data interpretation rules
     - formatting requirements
     - SQL escaping requirements
     - required output schema

7. **Connect**
   - `Merge` → `Code for Merge`
   - `Code for Merge` → `Gemini AI Studio`
   - `Gemini AI Studio` → `AI Agent`
   - `Google Gemini Chat Model` → `AI Agent` via AI language model connection

---

## 4.7 Build parsing and database persistence

1. **Add `Parse AI Agent output`**
   - Type: `Code`
   - Paste the original JS logic exactly.
   - This node:
     - strips markdown fences
     - parses JSON
     - reconstructs `provider_details`
     - returns `score`, `verdict_en`, `analysis_pl`, `provider_details`

2. **Add `Insert AI verdict`**
   - Type: `MySQL`
   - Operation: `Execute Query`
   - Enable always output data
   - Paste the original SQL exactly.

3. **Connect**
   - `AI Agent` → `Parse AI Agent output`
   - `Parse AI Agent output` → `Insert AI verdict`

---

## 4.8 Build alerting

1. **Add `Filter`**
   - Type: `Filter`
   - Condition:
     - Left value: `={{ $('Parse AI Agent output').item.json.score }}`
     - Operator: `>`
     - Right value: `5`

2. **Add `Get Email Pass`**
   - Type: `HashiCorp Vault`
   - Secret path: `cyber-sentinel/credentials/gmail/l94524506`

3. **Add `Send an Email`**
   - Type: `Send Email`
   - Configure SMTP credentials
   - Replace:
     - `toEmail: user@example.com`
     - `fromEmail: user@example.com`
   - Subject:
     ```text
     =Raport wygenerowany przez system Cyber Sentinel | Data: {{ new Date().toLocaleString('pl-PL') }}
     ```
   - HTML body: paste the original HTML template exactly if you want the same output formatting.

4. **Connect**
   - `Insert AI verdict` → `Filter`
   - `Filter` → `Get Email Pass`
   - `Get Email Pass` → `Send an Email`

---

## 4.9 Add sticky notes

To match the original layout, add these sticky notes:

1. `## Orchestration & Data Ingestion`
2. `## Multi-Source Threat Intelligence Enrichment`
3. `## Dual-Layer Persistence Strategy`
4. `## AI Synthesis & Cognitive Analysis`
5. `## Incident Reporting & Remediation`
6. Project note block containing purpose, setup instructions, documentation link, and author name

---

## 4.10 Important reproduction caveats

1. **Branch completion risk**
   - In the current design, only the “raw data persisted to MongoDB” branches feed the final `Merge`.
   - The “clear scan” branches stop in MySQL and do not produce normalized objects for the merge.
   - If one provider returns no data, the AI merge may not receive all three expected inputs.
   - To make the workflow more robust, create normalized no-data outputs for all providers and feed those into `Merge` too.

2. **Database consistency**
   - Ensure the dictionary tables contain:
     - `IP`
     - `FQDN`
     - `VirusTotal`
     - `Abuse_ThreatFox`
     - `Abuse_URLhaus`

3. **SQL safety**
   - The AI prompt attempts to enforce quote escaping, but relying on the model for SQL safety is fragile.
   - A safer rebuild would use prepared statements or parameterized inserts.

4. **Observable semantics**
   - ThreatFox and URLhaus branches query by FQDN but some Mongo payload builders store `observable_ip` as `resource` while labeling `type` as `FQDN`.
   - If rebuilding cleanly, consider storing the actual FQDN there.

5. **Email placeholders**
   - Replace the SMTP sender and recipient placeholders before activation.

6. **Naming consistency**
   - Keep exact node names if you reuse the provided expressions, including typos like:
     - `If query_status = ok - ThretFox`
     - `If query_status = ok - UrlHause`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Project: Cyber Sentinel | Visual project note in workflow |
| Purpose: Autonomous Cyber Threat Intelligence (CTI) & Passive DNS Monitoring. The workflow orchestrates DNS telemetry analysis using AI to detect malicious patterns and trigger autonomous response playbooks. | Project context |
| Setup: Ensure HashiCorp Vault is connected to handle API keys and credentials securely. | Infrastructure requirement |
| Setup: Configure the Ollama/Gemini node to enable bilingual threat assessment (EN/PL). | AI configuration note |
| Setup: Verify the connection to the PostgreSQL database (Neural Lake) for indicator enrichment. | Project note; however, the actual workflow uses MySQL nodes, so this note appears inconsistent with the implementation |
| Documentation | https://lukaszfd.github.io/cyber-sentinel/ |
| Author: Łukasz Dejko | Project credit |

## Additional implementation notes

- The workflow title in the JSON is **Automated Domain & IP Reputation Guard with HashiCorp Vault**, while the provided title says **Score DNS threats with VirusTotal, Abuse.ch, HashiCorp Vault and Gemini**. Both describe the same architecture, but the internal workflow name differs.
- The workflow is **active**.
- There are **no sub-workflows** or workflow-invoking nodes in this design.
- Multiple entry points are **not** present; the sole entry point is **Schedule Trigger**.