Automate cybersecurity threat analysis with GPT-4o, CVSS scoring and risk routing

https://n8nworkflows.xyz/workflows/automate-cybersecurity-threat-analysis-with-gpt-4o--cvss-scoring-and-risk-routing-14410


# Automate cybersecurity threat analysis with GPT-4o, CVSS scoring and risk routing

# 1. Workflow Overview

This workflow automates cybersecurity threat assessment using a multi-agent AI design in n8n. It starts from a manual trigger, sends a threat-analysis request to an orchestrator agent, delegates work to two specialist AI tool-agents, enforces a structured JSON output schema, and then routes the final result into different report formats depending on the overall risk rating.

Typical use cases include:

- SOC triage support
- Threat modeling and attack surface reviews
- STRIDE-based risk analysis
- CVSS-style scoring for attack scenarios
- Executive and operational reporting from a single analysis flow

## 1.1 Input Reception and Orchestration

A manual trigger starts the workflow. The main AI orchestrator receives either a provided `analysis_request` or a built-in default prompt describing a comprehensive threat-modeling task.

## 1.2 Threat Intelligence Analysis

One specialist agent focuses on analyzing security logs, authentication traces, and anomaly data. It can call:
- an HTTP tool to fetch logs from an internal API
- a calculator tool for risk-related numeric operations

## 1.3 Attack Surface Mapping and Risk Quantification

A second specialist agent performs attack-surface mapping and structured threat modeling. It can call:
- a STRIDE categorization code tool
- a CVSS-style scoring code tool

## 1.4 Structured Output Validation

The orchestrator is connected to a structured output parser requiring a predefined schema. This ensures the final AI output includes executive summary, threat intelligence, attack surface findings, SOC guidance, and overall risk rating in a machine-readable format.

## 1.5 Severity-Based Routing and Report Formatting

The parsed result is routed by `overall_risk_rating` through a Switch node:
- `Critical` → SOC alert format
- `High` → executive report format
- `Medium` → standard report format

There is no explicit rule for `Low`, so unmatched results may not be routed anywhere unless n8n default handling is added.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Trigger and Primary Orchestration

### Overview
This block starts the workflow and sends the initial analysis prompt to the central AI orchestrator. The orchestrator coordinates the full process, invokes the two specialist AI tools, and produces a structured final result.

### Nodes Involved
- Start Threat Analysis
- Cybersecurity Orchestrator Agent
- Orchestrator Chat Model
- Structured Threat Report Parser

### Node Details

#### Start Threat Analysis
- **Type and role:** `n8n-nodes-base.manualTrigger`; manual entry point for testing or on-demand execution.
- **Configuration choices:** No custom parameters; it starts when manually run in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Cybersecurity Orchestrator Agent`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or failures:** None beyond standard manual execution behavior.
- **Sub-workflow reference:** None.

#### Cybersecurity Orchestrator Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; primary AI coordinator.
- **Configuration choices:**
  - Prompt type is explicitly defined.
  - Input text uses:
    - `{{$json.analysis_request || 'Perform comprehensive threat modeling ...'}}`
  - System prompt instructs the agent to:
    1. delegate log analysis to the Threat Intelligence Agent
    2. delegate attack-surface and STRIDE work to the Attack Surface Mapping Agent
    3. synthesize findings for SOC and executive audiences
  - Output parser is enabled.
- **Key expressions or variables used:**
  - `{{$json.analysis_request || '...default request...'}}`
- **Input and output connections:**
  - Main input from `Start Threat Analysis`
  - AI language model input from `Orchestrator Chat Model`
  - AI tool inputs from:
    - `Threat Intelligence Agent`
    - `Attack Surface Mapping Agent`
  - AI output parser from `Structured Threat Report Parser`
  - Main output to `Route by Risk Severity`
- **Version-specific requirements:** Type version 3.1; requires compatible LangChain/AI nodes in the n8n instance.
- **Edge cases or failures:**
  - OpenAI credential failures via linked model node
  - malformed AI output if model fails to satisfy schema
  - incomplete delegation to tools if model chooses not to call them
  - token/context overrun if prompts or fetched logs are very large
- **Sub-workflow reference:** None; this is not an Execute Workflow node.

#### Orchestrator Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend for the orchestrator.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connects as AI language model to `Cybersecurity Orchestrator Agent`
- **Version-specific requirements:** Type version 1.3; requires OpenAI credentials and access to `gpt-4o`.
- **Edge cases or failures:**
  - invalid API key
  - insufficient model access
  - rate limiting
  - model timeout
- **Sub-workflow reference:** None.

#### Structured Threat Report Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces schema-constrained AI output.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `executive_summary`
    - `threat_intelligence`
    - `attack_surface`
    - `soc_operational_guidance`
    - `overall_risk_rating`
  - Nested objects include:
    - attack vectors with severity and recommendations
    - STRIDE analysis categories
    - lateral movement scenarios with CVSS scores
    - risk quantification and assets at risk
- **Key expressions or variables used:** Manual schema only.
- **Input and output connections:**
  - Connected as AI output parser to `Cybersecurity Orchestrator Agent`
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failures:**
  - model returns text that does not match schema
  - enum mismatch, especially for severity labels
  - numeric fields returned as strings
  - missing required properties
- **Sub-workflow reference:** None.

---

## 2.2 Block: Threat Intelligence Analysis

### Overview
This block equips the orchestrator with a specialist AI tool-agent focused on security telemetry analysis. The agent can fetch logs from an HTTP endpoint and perform calculations through a calculator tool.

### Nodes Involved
- Threat Intelligence Agent
- Threat Intelligence Chat Model
- Fetch Security Logs Tool
- Risk Score Calculator

### Node Details

#### Threat Intelligence Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; AI tool-agent callable by the orchestrator.
- **Configuration choices:**
  - Input text is provided dynamically by the orchestrator using:
    - `{{$fromAI('security_analysis_task', 'The specific security analysis task to perform, including which logs or traces to analyze and what threats to look for') }}`
  - System prompt defines expertise in:
    - suspicious pattern analysis
    - auth anomaly detection
    - threat classification
    - mitigation advice
- **Key expressions or variables used:**
  - `$fromAI('security_analysis_task', ...)`
- **Input and output connections:**
  - AI language model from `Threat Intelligence Chat Model`
  - AI tools:
    - `Fetch Security Logs Tool`
    - `Risk Score Calculator`
  - Exposed as AI tool to `Cybersecurity Orchestrator Agent`
- **Version-specific requirements:** Type version 3.
- **Edge cases or failures:**
  - agent may request missing/invalid tool arguments
  - hallucinated endpoint values
  - output may be too verbose or insufficiently evidenced
- **Sub-workflow reference:** None.

#### Threat Intelligence Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model backend for the threat-intelligence agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connects as AI language model to `Threat Intelligence Agent`
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failures:**
  - auth or quota issues
  - latency/rate limiting
- **Sub-workflow reference:** None.

#### Fetch Security Logs Tool
- **Type and role:** `n8n-nodes-base.httpRequestTool`; tool callable by the threat-intelligence agent to fetch telemetry.
- **Configuration choices:**
  - URL is generated from AI input:
    - `{{$fromAI('log_endpoint', 'The internal API endpoint to fetch security logs from (e.g., /api/security/logs, /api/auth/traces, /api/anomalies)', 'string', '<__PLACEHOLDER_VALUE__internal_security_api_endpoint__>') }}`
  - No additional options configured.
  - Tool description makes it available as a log-fetching capability.
- **Key expressions or variables used:**
  - `$fromAI('log_endpoint', ..., 'string', '<__PLACEHOLDER_VALUE__internal_security_api_endpoint__>')`
- **Input and output connections:**
  - Exposed as AI tool to `Threat Intelligence Agent`
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or failures:**
  - placeholder URL not replaced
  - missing authentication headers
  - internal API inaccessible from n8n host
  - non-JSON or oversized responses
  - TLS or DNS issues
- **Sub-workflow reference:** None.

#### Risk Score Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator`; generic calculation tool for numeric reasoning.
- **Configuration choices:** Default configuration; no explicit parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Exposed as AI tool to `Threat Intelligence Agent`
- **Version-specific requirements:** Type version 1.
- **Edge cases or failures:**
  - limited usefulness if the agent does not formulate numeric expressions clearly
  - may not match the sticky note claim about configurable thresholds; no threshold config is present in JSON
- **Sub-workflow reference:** None.

---

## 2.3 Block: Attack Surface Mapping and CVSS/STRIDE Analysis

### Overview
This block provides a second specialist AI tool-agent responsible for attack-surface assessment, STRIDE categorization, lateral movement modeling, and CVSS-style severity scoring.

### Nodes Involved
- Attack Surface Mapping Agent
- Attack Surface Chat Model
- STRIDE Analysis Tool
- CVSS Scoring Tool

### Node Details

#### Attack Surface Mapping Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; AI tool-agent callable by the orchestrator.
- **Configuration choices:**
  - Dynamic prompt input:
    - `{{$fromAI('attack_surface_task', 'The specific attack surface mapping task to perform, including network topology to analyze and STRIDE categories to focus on') }}`
  - System message defines:
    - topology graph construction
    - STRIDE analysis
    - lateral movement simulation
    - CVSS-style scoring
    - prioritization of critical assets and paths
- **Key expressions or variables used:**
  - `$fromAI('attack_surface_task', ...)`
- **Input and output connections:**
  - AI language model from `Attack Surface Chat Model`
  - AI tools:
    - `STRIDE Analysis Tool`
    - `CVSS Scoring Tool`
  - Exposed as AI tool to `Cybersecurity Orchestrator Agent`
- **Version-specific requirements:** Type version 3.
- **Edge cases or failures:**
  - agent may provide incomplete fields to code tools
  - threat modeling may become speculative if no concrete environment data is passed
  - outputs can drift from schema expectations if synthesis is weak upstream
- **Sub-workflow reference:** None.

#### Attack Surface Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model backend for attack-surface analysis.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connects as AI language model to `Attack Surface Mapping Agent`
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failures:**
  - auth, quota, latency, model-access issues
- **Sub-workflow reference:** None.

#### STRIDE Analysis Tool
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; JavaScript tool for STRIDE categorization.
- **Configuration choices:**
  - Reads `threat_description` from the first input item.
  - Uses keyword matching to populate:
    - spoofing
    - tampering
    - repudiation
    - information_disclosure
    - denial_of_service
    - elevation_of_privilege
  - Returns categorized arrays plus timestamp.
- **Key expressions or variables used:**
  - `const threat = $input.first().json.threat_description || "";`
- **Input and output connections:**
  - Exposed as AI tool to `Attack Surface Mapping Agent`
- **Version-specific requirements:** Type version 1.3; requires code execution enabled in environment.
- **Edge cases or failures:**
  - simplistic keyword matching can under-classify or over-classify threats
  - empty input leads to empty category arrays
  - no multi-threat batch handling
  - synonyms not covered by keyword list
- **Sub-workflow reference:** None.

#### CVSS Scoring Tool
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; JavaScript tool calculating a CVSS-style base score.
- **Configuration choices:**
  - Accepts input fields like:
    - `attack_vector`
    - `attack_complexity`
    - `privileges_required`
    - `user_interaction`
    - `scope`
    - `confidentiality_impact`
    - `integrity_impact`
    - `availability_impact`
  - Applies weighted calculations and maps output to severity bands:
    - None
    - Low
    - Medium
    - High
    - Critical
- **Key expressions or variables used:**
  - `const input = $input.first().json;`
  - fallback defaults favor a relatively severe network-based scenario
- **Input and output connections:**
  - Exposed as AI tool to `Attack Surface Mapping Agent`
- **Version-specific requirements:** Type version 1.3; requires code execution support.
- **Edge cases or failures:**
  - missing input fields default to potentially high-risk assumptions
  - this is CVSS-style, not a full standards-compliant CVSS v3.1 implementation
  - no validation of enum values beyond code branching
  - malformed inputs may silently collapse to fallback values
- **Sub-workflow reference:** None.

---

## 2.4 Block: Severity Routing

### Overview
This block branches the orchestrator’s final output into different downstream formatting paths according to the `overall_risk_rating` field.

### Nodes Involved
- Route by Risk Severity

### Node Details

#### Route by Risk Severity
- **Type and role:** `n8n-nodes-base.switch`; routes data into severity-specific branches.
- **Configuration choices:**
  - Rule 1: `{{$json.overall_risk_rating}} == "Critical"`
  - Rule 2: `{{$json.overall_risk_rating}} == "High"`
  - Rule 3: `{{$json.overall_risk_rating}} == "Medium"`
- **Key expressions or variables used:**
  - `{{$json.overall_risk_rating}}`
- **Input and output connections:**
  - Main input from `Cybersecurity Orchestrator Agent`
  - Output 0 → `Format SOC Alert`
  - Output 1 → `Format Executive Report`
  - Output 2 → `Format Standard Report`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or failures:**
  - `Low` has no matching rule
  - case-sensitive matching may fail for `critical`, `HIGH`, etc.
  - if parser output is nested under `output`, this node may not match depending on actual runtime structure
- **Sub-workflow reference:** None.

---

## 2.5 Block: Output Formatting

### Overview
This block converts the routed result into audience-specific payloads. Each Set node reshapes the data into a different reporting format.

### Nodes Involved
- Format SOC Alert
- Format Executive Report
- Format Standard Report

### Node Details

#### Format SOC Alert
- **Type and role:** `n8n-nodes-base.set`; prepares a critical alert payload for SOC operations.
- **Configuration choices:**
  - Sets fields such as:
    - `alert_type = CRITICAL_SECURITY_THREAT`
    - timestamp from `{{$now.toISO()}}`
    - risk rating and summary from `{{$json.output...}}`
  - Creates JSON-stringified values for actions, threat vectors, attack surface, and filtered immediate actions.
- **Key expressions or variables used:**
  - `{{$now.toISO()}}`
  - `{{$json.output.overall_risk_rating}}`
  - `{{$json.output.executive_summary}}`
  - `{{$json.output.soc_operational_guidance.filter(...)}}`
- **Input and output connections:**
  - Input from `Route by Risk Severity` critical branch
  - No downstream connection
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or failures:**
  - references `output.*`, but upstream switch uses top-level `overall_risk_rating`
  - if actual parser result is top-level, these expressions fail
  - fields declared as array/object are populated with `JSON.stringify(...)`, producing strings rather than native arrays/objects
- **Sub-workflow reference:** None.

#### Format Executive Report
- **Type and role:** `n8n-nodes-base.set`; prepares a higher-level executive report for high-risk cases.
- **Configuration choices:**
  - Sets:
    - `report_type = EXECUTIVE_CYBERSECURITY_POSTURE`
    - timestamp
    - risk rating from `{{$json.output.overall_risk_rating}}`
    - CVSS score
    - critical assets at risk
    - high priority actions
    - threat count
- **Key expressions or variables used:**
  - `{{$json.output.attack_surface.risk_quantification.overall_cvss_score}}`
  - `{{$json.output.threat_intelligence.emerging_attack_vectors.length}}`
- **Input and output connections:**
  - Input from `Route by Risk Severity` high branch
  - No downstream connection
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or failures:**
  - same possible `output` path mismatch as above
  - arrays are stringified into string values
  - missing `risk_quantification` or empty arrays can break expressions
- **Sub-workflow reference:** None.

#### Format Standard Report
- **Type and role:** `n8n-nodes-base.set`; builds a generic report for medium-risk cases.
- **Configuration choices:**
  - Sets:
    - `report_type = STANDARD_THREAT_ASSESSMENT`
    - timestamp
    - values from top-level fields like `{{$json.overall_risk_rating}}`
    - stringified full `threat_intelligence`, `attack_surface`, and `soc_operational_guidance`
- **Key expressions or variables used:**
  - `{{$json.overall_risk_rating}}`
  - `{{$json.executive_summary}}`
  - `{{JSON.stringify($json.threat_intelligence)}}`
- **Input and output connections:**
  - Input from `Route by Risk Severity` medium branch
  - No downstream connection
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or failures:**
  - unlike the other two formatter nodes, this one expects top-level fields, creating inconsistency
  - object/array fields are again converted to strings
- **Sub-workflow reference:** None.

---

## 2.6 Block: Documentation and In-Canvas Notes

### Overview
These sticky notes document intent, setup expectations, prerequisites, and visual grouping. They do not affect execution but are important for maintainers.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level description of the workflow’s purpose.
- **Configuration choices:** Large documentation note summarizing end-to-end architecture.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** Sticky note for setup guidance.
- **Configuration choices:** Documents model credentials, SIEM API, thresholds, STRIDE/CVSS setup, and routing thresholds.
- **Edge cases or failures:** Some guidance does not exactly match implemented logic; e.g. explicit CVSS thresholds are not configured in the Switch node.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and role:** Sticky note for prerequisites, use cases, customization, and benefits.
- **Configuration choices:** Includes implementation assumptions and one use case.
- **Edge cases or failures:** Mentions report template definitions but those are embodied as Set nodes rather than separate templates.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** Visual grouping note for trigger, threat intelligence, and scoring.
- **Configuration choices:** Explains why telemetry-based analysis matters.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** Visual grouping note for attack-surface analysis.
- **Configuration choices:** Explains STRIDE and CVSS purpose.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and role:** Visual grouping note for parse-and-route stage.
- **Configuration choices:** Explains structured extraction and severity routing.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and role:** Visual grouping note for report formatting.
- **Configuration choices:** Explains output targeting by audience.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Threat Analysis | n8n-nodes-base.manualTrigger | Manual entry point |  | Cybersecurity Orchestrator Agent | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Cybersecurity Orchestrator Agent | @n8n/n8n-nodes-langchain.agent | Main coordinator agent | Start Threat Analysis; Orchestrator Chat Model; Threat Intelligence Agent; Attack Surface Mapping Agent; Structured Threat Report Parser | Route by Risk Severity | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Orchestrator Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for orchestrator |  | Cybersecurity Orchestrator Agent | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Structured Threat Report Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Schema-enforced final output parser |  | Cybersecurity Orchestrator Agent | ## Parse & Route by Severity<br>**What:** Structured Threat Report Parser extracts findings; Rules router directs output by risk level.<br>**Why:** Ensures outputs are structured and stakeholder-appropriate without manual triage. |
| Threat Intelligence Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist tool-agent for log and anomaly analysis | Threat Intelligence Chat Model; Fetch Security Logs Tool; Risk Score Calculator | Cybersecurity Orchestrator Agent | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Threat Intelligence Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for threat-intelligence agent |  | Threat Intelligence Agent | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Fetch Security Logs Tool | n8n-nodes-base.httpRequestTool | Fetches security logs from internal API |  | Threat Intelligence Agent | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Risk Score Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Numeric calculation tool for threat analysis |  | Threat Intelligence Agent | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Attack Surface Mapping Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist tool-agent for attack-surface and STRIDE analysis | Attack Surface Chat Model; STRIDE Analysis Tool; CVSS Scoring Tool | Cybersecurity Orchestrator Agent | ## Attack Surface Mapping<br>**What:** Attack Surface Mapping Agent applies STRIDE methodology and CVSS scoring.<br>**Why:** Systematically identifies exploitable vectors and assigns industry-standard severity ratings. |
| Attack Surface Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for attack-surface agent |  | Attack Surface Mapping Agent | ## Attack Surface Mapping<br>**What:** Attack Surface Mapping Agent applies STRIDE methodology and CVSS scoring.<br>**Why:** Systematically identifies exploitable vectors and assigns industry-standard severity ratings. |
| STRIDE Analysis Tool | @n8n/n8n-nodes-langchain.toolCode | Keyword-based STRIDE categorization tool |  | Attack Surface Mapping Agent | ## Attack Surface Mapping<br>**What:** Attack Surface Mapping Agent applies STRIDE methodology and CVSS scoring.<br>**Why:** Systematically identifies exploitable vectors and assigns industry-standard severity ratings. |
| CVSS Scoring Tool | @n8n/n8n-nodes-langchain.toolCode | CVSS-style scoring code tool |  | Attack Surface Mapping Agent | ## Attack Surface Mapping<br>**What:** Attack Surface Mapping Agent applies STRIDE methodology and CVSS scoring.<br>**Why:** Systematically identifies exploitable vectors and assigns industry-standard severity ratings. |
| Route by Risk Severity | n8n-nodes-base.switch | Severity-based branching | Cybersecurity Orchestrator Agent | Format SOC Alert; Format Executive Report; Format Standard Report | ## Parse & Route by Severity<br>**What:** Structured Threat Report Parser extracts findings; Rules router directs output by risk level.<br>**Why:** Ensures outputs are structured and stakeholder-appropriate without manual triage. |
| Format SOC Alert | n8n-nodes-base.set | Formats critical-risk SOC alert payload | Route by Risk Severity |  | ## Format & Deliver Report<br>**What:** Generates SOC Alert, Executive Report, or Standard Report based on severity routing.<br>**Why:** Delivers the right level of detail to the right audience — operational, strategic, or routine. |
| Format Executive Report | n8n-nodes-base.set | Formats executive report for high-risk cases | Route by Risk Severity |  | ## Format & Deliver Report<br>**What:** Generates SOC Alert, Executive Report, or Standard Report based on severity routing.<br>**Why:** Delivers the right level of detail to the right audience — operational, strategic, or routine. |
| Format Standard Report | n8n-nodes-base.set | Formats standard report for medium-risk cases | Route by Risk Severity |  | ## Format & Deliver Report<br>**What:** Generates SOC Alert, Executive Report, or Standard Report based on severity routing.<br>**Why:** Delivers the right level of detail to the right audience — operational, strategic, or routine. |
| Sticky Note | n8n-nodes-base.stickyNote | In-canvas documentation |  |  | ## How It Works<br>This workflow automates end-to-end cybersecurity threat analysis using a multi-agent AI architecture, targeting Security Operations Centre (SOC) analysts, security engineers, and IT risk teams responsible for continuous threat monitoring and incident response. The core problem it solves is the slow, fragmented process of manually correlating threat intelligence, scoring vulnerabilities, and producing actionable reports, tasks that demand both speed and consistency under pressure. A manual trigger initiates the Cybersecurity Orchestrator Agent, which coordinates two specialist sub-agents: a Threat Intelligence Agent (backed by security log fetching and risk scoring tools) and an Attack Surface Mapping Agent (leveraging STRIDE analysis and CVSS scoring tools). Each agent operates with its own chat model and memory. Outputs are parsed by a Structured Threat Report Parser, then routed by a Rules-based Risk Severity router into three report formats such as SOC Alert, Executive Report, or Standard Report, ensuring every threat is communicated at the right level of urgency to the right audience. |
| Sticky Note1 | n8n-nodes-base.stickyNote | In-canvas setup guidance |  |  | ## Setup Steps<br>1. Connect your LLM API credentials to all Chat Model nodes (Orchestrator, Threat Intelligence, Attack Surface).<br>2. Configure the Fetch Security Logs Tool with your SIEM or log source API credentials.<br>3. Set risk threshold rules in the Risk Score Calculator node.<br>4. Define STRIDE and CVSS parameters in their respective tool nodes.<br>5. Set routing thresholds (e.g., CVSS ≥9 → SOC Alert, ≥6 → Executive, <6 → Standard) in Route by Risk Severity. |
| Sticky Note2 | n8n-nodes-base.stickyNote | In-canvas prerequisites and benefits |  |  | ## Prerequisites<br>- LLM API key (OpenAI or compatible)<br>- SIEM or security log source with API access<br>- CVSS and STRIDE configuration parameters<br>- Report template definitions for each severity tier<br>## Use Cases<br>- Auto-triage incoming vulnerability disclosures into severity-ranked reports.<br>## Customisation<br>- Add more routing branches (e.g., Critical, Zero-Day).<br>## Benefits<br>- Accelerates threat triage from hours to minutes. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual note for trigger and threat intelligence area |  |  | ## Trigger, Threat Intelligence & Risk Scoring<br>**What:** Threat Intelligence Agent fetches security logs and calculates risk scores.<br>**Why:** Grounds AI analysis in real telemetry data, enabling evidence-based risk prioritisation. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual note for attack surface area |  |  | ## Attack Surface Mapping<br>**What:** Attack Surface Mapping Agent applies STRIDE methodology and CVSS scoring.<br>**Why:** Systematically identifies exploitable vectors and assigns industry-standard severity ratings. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual note for parse and route area |  |  | ## Parse & Route by Severity<br>**What:** Structured Threat Report Parser extracts findings; Rules router directs output by risk level.<br>**Why:** Ensures outputs are structured and stakeholder-appropriate without manual triage. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual note for formatting area |  |  | ## Format & Deliver Report<br>**What:** Generates SOC Alert, Executive Report, or Standard Report based on severity routing.<br>**Why:** Delivers the right level of detail to the right audience — operational, strategic, or routine. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI agent for cybersecurity threat analysis with CVSS scoring and risk routing`.

2. **Add the trigger node**
   - Add a **Manual Trigger** node.
   - Name it `Start Threat Analysis`.

3. **Add the main orchestrator agent**
   - Add an **AI Agent** node of type `@n8n/n8n-nodes-langchain.agent`.
   - Name it `Cybersecurity Orchestrator Agent`.
   - Set **Prompt Type** to defined/manual prompt mode.
   - Set the main input text to:
     - `{{$json.analysis_request || 'Perform comprehensive threat modeling and attack surface analysis of our current security posture. Analyze internal security logs, authentication traces, and anomaly detection outputs to identify emerging threats. Construct network topology models and assess lateral movement risks using STRIDE methodology with CVSS-style scoring.' }}`
   - Add the long system message describing the orchestrator’s role and delegation behavior.
   - Enable structured output parsing.

4. **Connect the trigger to the orchestrator**
   - Connect `Start Threat Analysis` → `Cybersecurity Orchestrator Agent`.

5. **Add the orchestrator model**
   - Add an **OpenAI Chat Model** node.
   - Name it `Orchestrator Chat Model`.
   - Choose model `gpt-4o`.
   - Set temperature to `0.3`.
   - Connect OpenAI credentials.
   - Connect it to the orchestrator using the **AI Language Model** connection.

6. **Add the structured output parser**
   - Add a **Structured Output Parser** node.
   - Name it `Structured Threat Report Parser`.
   - Set schema type to manual.
   - Paste a schema defining:
     - `executive_summary` as string
     - `threat_intelligence` object with:
       - `emerging_attack_vectors` array of objects
       - `authentication_anomalies` array
       - `log_analysis_summary` string
     - `attack_surface` object with:
       - `network_topology`
       - `stride_analysis`
       - `lateral_movement_scenarios`
       - `risk_quantification`
     - `soc_operational_guidance` array of objects
     - `overall_risk_rating` enum: `Critical`, `High`, `Medium`, `Low`
   - Mark the main top-level fields as required.
   - Connect it to the orchestrator as **AI Output Parser**.

7. **Add the Threat Intelligence specialist agent**
   - Add an **AI Agent Tool** node.
   - Name it `Threat Intelligence Agent`.
   - Set its text input to:
     - `{{$fromAI('security_analysis_task', 'The specific security analysis task to perform, including which logs or traces to analyze and what threats to look for') }}`
   - Add the system message focused on:
     - log analysis
     - authentication irregularities
     - anomaly detection interpretation
     - attack-vector classification
     - mitigation recommendations
   - Add a clear tool description explaining this agent analyzes internal logs and returns threat intelligence.
   - Connect this node to the orchestrator as an **AI Tool**.

8. **Add the Threat Intelligence model**
   - Add an **OpenAI Chat Model** node.
   - Name it `Threat Intelligence Chat Model`.
   - Set model to `gpt-4o`.
   - Set temperature to `0.2`.
   - Use the same or another OpenAI credential.
   - Connect it to `Threat Intelligence Agent` as **AI Language Model**.

9. **Add the security log fetch tool**
   - Add an **HTTP Request Tool** node.
   - Name it `Fetch Security Logs Tool`.
   - Set the URL to:
     - `{{$fromAI('log_endpoint', 'The internal API endpoint to fetch security logs from (e.g., /api/security/logs, /api/auth/traces, /api/anomalies)', 'string', '<__PLACEHOLDER_VALUE__internal_security_api_endpoint__>') }}`
   - Add a tool description indicating it fetches security logs and anomaly outputs.
   - If your SIEM or log source requires auth:
     - configure headers, bearer token, basic auth, or OAuth2 as appropriate
   - Prefer returning JSON.
   - Connect it to `Threat Intelligence Agent` as **AI Tool**.

10. **Add the calculator tool**
    - Add a **Calculator Tool** node.
    - Name it `Risk Score Calculator`.
    - Keep default configuration unless you want custom prompt instructions around numeric calculations.
    - Connect it to `Threat Intelligence Agent` as **AI Tool**.

11. **Add the Attack Surface specialist agent**
    - Add an **AI Agent Tool** node.
    - Name it `Attack Surface Mapping Agent`.
    - Set text input to:
      - `{{$fromAI('attack_surface_task', 'The specific attack surface mapping task to perform, including network topology to analyze and STRIDE categories to focus on') }}`
    - Add the system message focused on:
      - network topology mapping
      - trust boundaries
      - STRIDE threat modeling
      - lateral movement scenarios
      - CVSS-style scoring
    - Add a tool description explaining its outputs.
    - Connect it to `Cybersecurity Orchestrator Agent` as **AI Tool**.

12. **Add the Attack Surface model**
    - Add an **OpenAI Chat Model** node.
    - Name it `Attack Surface Chat Model`.
    - Set model to `gpt-4o`.
    - Set temperature to `0.2`.
    - Connect OpenAI credentials.
    - Connect it to `Attack Surface Mapping Agent` as **AI Language Model**.

13. **Add the STRIDE code tool**
    - Add a **Code Tool** node for AI tools.
    - Name it `STRIDE Analysis Tool`.
    - Paste JavaScript that:
      - reads `threat_description`
      - creates arrays for the six STRIDE categories
      - performs keyword matching
      - returns categorized results with timestamp
    - Add a description explaining expected input and output.
    - Connect it to `Attack Surface Mapping Agent` as **AI Tool**.

14. **Add the CVSS code tool**
    - Add another **Code Tool** node.
    - Name it `CVSS Scoring Tool`.
    - Paste JavaScript that:
      - reads CVSS-style fields from the input JSON
      - computes exploitability and impact
      - calculates a 0–10 base score
      - derives a severity label
      - returns the score, severity, and parameter echo
    - Add a description explaining required input fields.
    - Connect it to `Attack Surface Mapping Agent` as **AI Tool**.

15. **Add the risk router**
    - Add a **Switch** node.
    - Name it `Route by Risk Severity`.
    - Configure three rules based on `{{$json.overall_risk_rating}}`:
      1. equals `Critical`
      2. equals `High`
      3. equals `Medium`
   - Connect `Cybersecurity Orchestrator Agent` → `Route by Risk Severity`.

16. **Add the critical SOC formatter**
    - Add a **Set** node.
    - Name it `Format SOC Alert`.
    - Create fields:
      - `alert_type` = `CRITICAL_SECURITY_THREAT`
      - `timestamp` = `{{$now.toISO()}}`
      - `risk_rating` = `{{$json.output.overall_risk_rating}}`
      - `executive_summary` = `{{$json.output.executive_summary}}`
      - `soc_actions` = `{{JSON.stringify($json.output.soc_operational_guidance)}}`
      - `threat_vectors` = `{{JSON.stringify($json.output.threat_intelligence.emerging_attack_vectors)}}`
      - `attack_surface` = `{{JSON.stringify($json.output.attack_surface)}}`
      - `immediate_actions_required` = `{{JSON.stringify($json.output.soc_operational_guidance.filter(action => action.priority === 'P0-Critical' || action.priority === 'P1-High'))}}`
    - Connect from the `Critical` output of the Switch node.

17. **Add the executive formatter**
    - Add another **Set** node.
    - Name it `Format Executive Report`.
    - Create fields:
      - `report_type` = `EXECUTIVE_CYBERSECURITY_POSTURE`
      - `timestamp` = `{{$now.toISO()}}`
      - `risk_rating` = `{{$json.output.overall_risk_rating}}`
      - `executive_summary` = `{{$json.output.executive_summary}}`
      - `overall_cvss_score` = `{{$json.output.attack_surface.risk_quantification.overall_cvss_score}}`
      - `critical_assets_at_risk` = `{{JSON.stringify($json.output.attack_surface.risk_quantification.critical_assets_at_risk)}}`
      - `high_priority_actions` = `{{JSON.stringify($json.output.soc_operational_guidance.filter(action => action.priority === 'P0-Critical' || action.priority === 'P1-High').map(action => action.action))}}`
      - `threat_count` = `{{$json.output.threat_intelligence.emerging_attack_vectors.length}}`
    - Connect from the `High` output of the Switch node.

18. **Add the standard formatter**
    - Add a third **Set** node.
    - Name it `Format Standard Report`.
    - Create fields:
      - `report_type` = `STANDARD_THREAT_ASSESSMENT`
      - `timestamp` = `{{$now.toISO()}}`
      - `risk_rating` = `{{$json.overall_risk_rating}}`
      - `executive_summary` = `{{$json.executive_summary}}`
      - `threat_intelligence` = `{{JSON.stringify($json.threat_intelligence)}}`
      - `attack_surface` = `{{JSON.stringify($json.attack_surface)}}`
      - `soc_operational_guidance` = `{{JSON.stringify($json.soc_operational_guidance)}}`
    - Connect from the `Medium` output of the Switch node.

19. **Optionally add a Low/default route**
    - Recommended improvement:
      - add a fourth Switch rule for `Low`
      - or enable/funnel unmatched items into a fallback formatter
   - As provided, low-risk results are not explicitly handled.

20. **Add sticky notes**
    - Create seven sticky notes if you want the same visual documentation:
      - workflow explanation
      - setup steps
      - prerequisites/use cases/customization/benefits
      - trigger/threat intelligence area
      - attack surface area
      - parse/route area
      - format/deliver area

21. **Configure credentials**
    - **OpenAI credentials:** required on all three Chat Model nodes.
    - **HTTP/API credentials for logs:** required on `Fetch Security Logs Tool` if your endpoint is protected.
    - Ensure the n8n host can reach the internal security API over the network.

22. **Test the workflow**
    - Run manually.
    - Optionally feed input JSON to the trigger context containing:
      - `analysis_request`
    - Verify:
      - orchestrator calls both tool-agents
      - final output satisfies parser schema
      - switch routes properly
      - Set node expressions match actual output structure

23. **Recommended hardening before production**
    - Normalize output paths so all formatters use either top-level fields or `output.*`, but not both.
    - Replace `JSON.stringify(...)` with native arrays/objects if downstream systems expect structured JSON.
    - Add HTTP auth, retry logic, and timeouts on the log fetch tool.
    - Add explicit handling for `Low` risk.
    - Consider truncating or summarizing very large log responses before model use.

### Sub-workflow setup
This workflow does **not** use Execute Workflow or Sub-Workflow nodes. The “sub-agents” are AI tool-agents within the same workflow, not separate n8n workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end cybersecurity threat analysis using a multi-agent AI architecture, targeting SOC analysts, security engineers, and IT risk teams. | In-canvas workflow description |
| Connect your LLM API credentials to all Chat Model nodes. | Setup guidance |
| Configure the Fetch Security Logs Tool with your SIEM or log source API credentials. | Setup guidance |
| Set risk threshold rules in the Risk Score Calculator node. | Setup guidance; note that no explicit threshold configuration is present in the current Calculator node |
| Define STRIDE and CVSS parameters in their respective tool nodes. | Setup guidance |
| Set routing thresholds such as CVSS ≥9 → SOC Alert, ≥6 → Executive, <6 → Standard. | Setup guidance; note that current Switch node routes by `overall_risk_rating`, not by numeric CVSS |
| Prerequisites: LLM API key, SIEM/log source API access, CVSS and STRIDE parameters, report template definitions. | In-canvas prerequisites |
| Use case: Auto-triage incoming vulnerability disclosures into severity-ranked reports. | In-canvas use case |
| Customization: Add more routing branches such as Critical or Zero-Day. | In-canvas customization |
| Benefit: Accelerates threat triage from hours to minutes. | In-canvas benefits |

## Additional implementation observations
- The workflow title in metadata is `AI agent for cybersecurity threat analysis with CVSS scoring and risk routing`, while the provided external title is slightly different.
- There is a structural inconsistency in formatter expressions:
  - `Format SOC Alert` and `Format Executive Report` expect `output.*`
  - `Format Standard Report` expects top-level fields
- If the orchestrator returns parsed data at the top level, the first two formatter nodes will likely fail unless adjusted.
- The current switch logic handles only `Critical`, `High`, and `Medium`. `Low` outputs are not routed.
- The HTTP log tool currently uses an AI-provided endpoint with a placeholder default and no authentication/options configured; this must be completed for real use.