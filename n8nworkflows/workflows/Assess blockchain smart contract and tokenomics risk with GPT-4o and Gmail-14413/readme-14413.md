Assess blockchain smart contract and tokenomics risk with GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/assess-blockchain-smart-contract-and-tokenomics-risk-with-gpt-4o-and-gmail-14413


# Assess blockchain smart contract and tokenomics risk with GPT-4o and Gmail

# 1. Workflow Overview

This workflow performs an AI-assisted blockchain risk assessment focused on two complementary dimensions:

- **Smart contract security risk** from Solidity source code
- **Tokenomics and DAO sustainability risk** using simulation-based economic analysis

It is designed for teams that want an automated first-pass review of a blockchain project before deployment, audit escalation, governance launch, or investment review. The workflow reads a Solidity file from disk, submits the content to a coordinating AI agent, delegates specialized analysis to two sub-agents, converts the result into a structured risk report, stores key fields in a data table, and emails a summary through Gmail.

## 1.1 Input Reception

The workflow starts manually and reads Solidity source code from a local file path. This creates the raw text input that drives the downstream analysis.

## 1.2 Multi-Agent Risk Orchestration

A central AI agent, the **Blockchain Risk Orchestrator**, receives the Solidity content and coordinates two specialist tools:

- **Smart Contract Auditor Agent**
- **Tokenomics Sustainability Agent**

The orchestrator is backed by GPT-4o and constrained by a structured output parser so the final result conforms to a defined JSON schema.

## 1.3 Smart Contract Static Analysis

The smart contract branch uses a dedicated AI agent with a lower-temperature GPT-4o model plus two tools:

- A custom **Static Analysis Tool** implemented in JavaScript for regex-based vulnerability detection
- A **Risk Score Calculator** tool for numerical reasoning

This block is responsible for code-level vulnerability findings such as reentrancy, unchecked external calls, overflow risk, and weak access control.

## 1.4 Tokenomics Simulation and Governance Risk Analysis

The tokenomics branch uses another dedicated AI agent with GPT-4o and two tools:

- A custom **Monte Carlo Simulation Tool**
- The shared **Risk Score Calculator**

This block estimates DAO stability, supply/inflation dynamics, governance concentration, participation, and centralization metrics.

## 1.5 Report Structuring, Storage, and Notification

The orchestrator’s output is parsed into a structured schema, transformed into storage-friendly fields, written into an n8n Data Table, and summarized in a Gmail message to stakeholders.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block initializes the workflow and loads Solidity source code from the local filesystem. The content of the file becomes the primary input for the AI orchestration layer.

### Nodes Involved
- Start Analysis
- Read Solidity Files

### Node Details

#### Start Analysis
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for testing or ad hoc execution.
- **Configuration choices:** No parameters are configured; execution begins when triggered manually in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Read Solidity Files`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - No runtime issue by itself
  - Not suitable for unattended production execution unless replaced with a scheduled or event-based trigger
- **Sub-workflow reference:** None

#### Read Solidity Files
- **Type and technical role:** `n8n-nodes-base.readWriteFile`  
  Reads a file from disk and outputs its contents.
- **Configuration choices:**  
  - Uses `fileSelector` set to a placeholder path: `<__PLACEHOLDER_VALUE__path_to_solidity_file__>`
  - No extra options configured
- **Key expressions or variables used:**  
  - Downstream, the workflow expects the file content in `{{$json.data}}`
- **Input and output connections:**  
  - Input: `Start Analysis`
  - Output: `Blockchain Risk Orchestrator`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - File path invalid or inaccessible
  - n8n runtime lacks filesystem access
  - File encoding issues
  - Large Solidity files may lead to token limits later in the LLM chain
  - If the file is binary or returned differently in some environments, `data` may not be populated as expected
- **Sub-workflow reference:** None

---

## 2.2 Multi-Agent Risk Orchestration

### Overview
This block coordinates the entire analysis. The orchestrator agent receives Solidity code, delegates sub-tasks to two specialist agents, and returns a structured final report constrained by a schema.

### Nodes Involved
- Blockchain Risk Orchestrator
- Orchestrator Model
- Structured Risk Report
- Smart Contract Auditor Agent
- Tokenomics Sustainability Agent

### Node Details

#### Blockchain Risk Orchestrator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI agent that interprets the Solidity file and decides when to call specialist tool-agents.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Main text input: `={{ $json.data }}`
  - System message instructs the agent to:
    - coordinate specialized agents
    - delegate smart contract analysis to the Auditor Agent
    - delegate tokenomics sustainability analysis to the Tokenomics Agent
    - aggregate findings into a comprehensive risk assessment with remediation plans
  - Output parser enabled
- **Key expressions or variables used:**  
  - `{{$json.data}}` from the file-reading node
- **Input and output connections:**  
  - Main input: `Read Solidity Files`
  - AI language model input: `Orchestrator Model`
  - AI tool inputs: `Smart Contract Auditor Agent`, `Tokenomics Sustainability Agent`
  - AI output parser input: `Structured Risk Report`
  - Main output: `Prepare Storage Data`
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases or potential failure types:**  
  - LLM may not call both specialist agents reliably unless the prompt is strong enough
  - If tokenomics data is absent from the Solidity file, the tokenomics sub-agent may infer values or rely on defaults, reducing realism
  - Large contract source may exceed LLM context limits
  - Structured output parser may fail if the agent returns malformed or incomplete schema output
  - Tool-call planning can vary by model behavior
- **Sub-workflow reference:** None; this is not an Execute Workflow node, but it orchestrates tool-agents.

#### Orchestrator Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o chat model used by the orchestrator.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - No built-in tools enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output to `Blockchain Risk Orchestrator`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - API quota exhaustion
  - Model availability changes
  - Latency or timeout issues on large prompts
- **Sub-workflow reference:** None

#### Structured Risk Report
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured JSON output schema on the orchestrator’s final response.
- **Configuration choices:**  
  - Manual JSON schema
  - Required top-level fields:
    - `executiveSummary`
    - `smartContractFindings`
    - `tokenomicsFindings`
    - `remediationPlan`
    - `overallRiskRating`
  - Schema includes nested arrays and objects for vulnerability groups, scores, tokenomics metrics, and action plans
- **Key expressions or variables used:** None directly; schema validation shapes downstream data access.
- **Input and output connections:**  
  - Connected as AI output parser to `Blockchain Risk Orchestrator`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Parsing errors if the model omits required fields
  - Type mismatch, for example returning strings instead of numbers
  - Missing nested arrays can break later expressions such as `.length`
- **Sub-workflow reference:** None

#### Smart Contract Auditor Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  A specialist AI tool-agent called by the orchestrator for Solidity security review.
- **Configuration choices:**  
  - Input text retrieved via AI variable:
    - `={{ $fromAI('solidity_code', 'The Solidity smart contract source code to analyze', 'string') }}`
  - System message instructs it to analyze:
    - reentrancy
    - integer overflow/underflow for pre-0.8 Solidity
    - unchecked external calls
    - access control flaws
    - logic errors and edge cases
  - Tool description clearly frames it as a static analysis specialist
- **Key expressions or variables used:**  
  - `$fromAI('solidity_code', ...)`
- **Input and output connections:**  
  - AI language model input: `Auditor Model`
  - AI tool inputs: `Static Analysis Tool`, `Risk Score Calculator`
  - AI tool output to `Blockchain Risk Orchestrator`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - The orchestrator may fail to pass an adequate `solidity_code` value
  - The agent may hallucinate line references beyond the regex tool output
  - If the tool receives no `code` field, static analysis will run on an empty string
- **Sub-workflow reference:** None

#### Tokenomics Sustainability Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool-agent for tokenomics, governance concentration, and sustainability analysis.
- **Configuration choices:**  
  - Input text retrieved via AI variable:
    - `={{ $fromAI('tokenomics_data', 'Token economics parameters including supply, distribution, governance rules, and emission schedules', 'string') }}`
  - System message instructs it to analyze:
    - token supply dynamics
    - inflation scenarios using Monte Carlo simulations
    - governance centralization risks
    - economic sustainability
    - DAO stability metrics such as Gini and Nakamoto coefficients
  - Tool description frames it as a simulation and metrics engine
- **Key expressions or variables used:**  
  - `$fromAI('tokenomics_data', ...)`
- **Input and output connections:**  
  - AI language model input: `Tokenomics Model`
  - AI tool inputs: `Monte Carlo Simulation Tool`, `Risk Score Calculator`
  - AI tool output to `Blockchain Risk Orchestrator`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - There is no dedicated upstream tokenomics dataset in the workflow; the orchestrator must infer or synthesize tokenomics input
  - Results may rely heavily on default simulation values if no real tokenomics parameters are supplied
  - Model may produce polished but weakly grounded conclusions
- **Sub-workflow reference:** None

---

## 2.3 Smart Contract Static Analysis

### Overview
This block performs pattern-based Solidity vulnerability detection and numeric risk reasoning. It supports the Smart Contract Auditor Agent with deterministic tool outputs and calculator support.

### Nodes Involved
- Auditor Model
- Static Analysis Tool
- Risk Score Calculator

### Node Details

#### Auditor Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model dedicated to the smart contract agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
  - No built-in tools
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output to `Smart Contract Auditor Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid or missing OpenAI credentials
  - Model context issues with very large contracts
  - Inconsistent interpretation of static-analysis findings
- **Sub-workflow reference:** None

#### Static Analysis Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  Custom JavaScript tool that scans Solidity source code with regex-style checks.
- **Configuration choices:**  
  - Reads:
    - `code` from `$input.first().json.code || ''`
    - `analysisType` from `$input.first().json.analysisType || 'full'`
  - Detects:
    - Reentrancy by identifying external calls followed by nearby state changes
    - Unchecked `.call{}` usages
    - Overflow risks in Solidity versions `<0.8.0`
    - Missing access control on public functions with sensitive names
  - Returns:
    - `findings`
    - `totalIssues`
    - `analysisComplete`
- **Key expressions or variables used:**  
  - `$input.first().json.code`
  - `$input.first().json.analysisType`
- **Input and output connections:**  
  - AI tool output to `Smart Contract Auditor Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - False positives from regex matching
  - False negatives on complex Solidity patterns
  - No actual AST parsing despite description suggesting AST-like behavior
  - `RegExp.test()` with global regex flags can be stateful and cause inconsistent matches across loop iterations
  - Access-control detection is simplistic and may miss modifier use on multiline declarations
  - If `code` is absent, tool silently analyzes an empty string
- **Sub-workflow reference:** None

#### Risk Score Calculator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  Shared calculator for arithmetic/statistical reasoning by both specialist agents.
- **Configuration choices:**  
  - No explicit parameters
- **Key expressions or variables used:** None in node configuration.
- **Input and output connections:**  
  - AI tool output to:
    - `Smart Contract Auditor Agent`
    - `Tokenomics Sustainability Agent`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Limited to calculator-style reasoning; not a full analytics engine
  - If prompts do not clearly specify formulas, the model may underuse it
- **Sub-workflow reference:** None

---

## 2.4 Tokenomics Simulation and Governance Risk Analysis

### Overview
This block models token supply and governance risk under uncertainty. It supports the Tokenomics Sustainability Agent with simulation output and numerical calculation.

### Nodes Involved
- Tokenomics Model
- Monte Carlo Simulation Tool
- Risk Score Calculator

### Node Details

#### Tokenomics Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model dedicated to tokenomics analysis.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
  - No built-in tools
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output to `Tokenomics Sustainability Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Authentication or quota issues
  - Weak grounding if real tokenomics parameters are unavailable
- **Sub-workflow reference:** None

#### Monte Carlo Simulation Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  Custom JavaScript tool performing probabilistic tokenomics and governance simulations.
- **Configuration choices:**  
  - Reads raw parameters from `$input.first().json`
  - Defaults:
    - `simulations`: `10000`
    - `timeHorizon`: `365`
    - `initialSupply`: `1+1234567890`
    - `inflationRate`: `0.05`
    - `distributionGini`: `0.7`
    - `topHoldersPercent`: `0.3`
  - Produces:
    - `supplyProjections`
    - `inflationScenarios`
    - `centralizationRisks`
    - `stabilityMetrics`
  - Calculates:
    - means
    - standard deviation
    - percentiles
    - DAO stability score
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - AI tool output to `Tokenomics Sustainability Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - The default `initialSupply` expression `1+1234567890` evaluates numerically to `1234567891`; this is likely accidental and may not reflect intended supply
  - Running 10,000 simulations can be CPU-intensive in constrained n8n environments
  - Uses very simplified assumptions for governance concentration and supply growth
  - No input validation for negative rates or invalid percentages
  - Arrays can become large and may increase memory usage
- **Sub-workflow reference:** None

---

## 2.5 Report Structuring, Storage, and Notification

### Overview
This block converts the orchestrator output into storage fields, saves the result to an n8n data table, and sends an email summary through Gmail. It turns the analysis into an auditable and shareable deliverable.

### Nodes Involved
- Prepare Storage Data
- Store Risk Analysis
- Send Risk Report Email

### Node Details

#### Prepare Storage Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes and enriches the final structured report with storage-oriented fields.
- **Configuration choices:**  
  - Adds:
    - `timestamp = {{$now.toISO()}}`
    - `overallRiskRating = {{$json.overallRiskRating}}`
    - `securityScore = {{$json.smartContractFindings.overallSecurityScore}}`
    - `daoStabilityScore = {{$json.tokenomicsFindings.daoStabilityScore}}`
    - `criticalVulnerabilities = {{$json.smartContractFindings.criticalVulnerabilities.length}}`
    - `executiveSummary = {{$json.executiveSummary}}`
  - `includeOtherFields` is enabled, so original fields remain available
- **Key expressions or variables used:**  
  - `$now.toISO()`
  - `$json.overallRiskRating`
  - `$json.smartContractFindings.overallSecurityScore`
  - `$json.tokenomicsFindings.daoStabilityScore`
  - `$json.smartContractFindings.criticalVulnerabilities.length`
  - `$json.executiveSummary`
- **Input and output connections:**  
  - Input: `Blockchain Risk Orchestrator`
  - Output: `Store Risk Analysis`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If the structured parser output is incomplete, expressions like `.length` may fail
  - Numeric fields may be null if the LLM does not produce valid numbers
- **Sub-workflow reference:** None

#### Store Risk Analysis
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists the processed risk record into an n8n Data Table.
- **Configuration choices:**  
  - Mapping mode: auto-map input data
  - Data table ID placeholder:
    - `<__PLACEHOLDER_VALUE__risk_analysis_table__>`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - Input: `Prepare Storage Data`
  - Output: `Send Risk Report Email`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Data Table feature availability depends on n8n version and plan/setup
  - Invalid table ID
  - Schema mismatch if auto-mapped fields do not match table columns
  - Storage permission issues
- **Sub-workflow reference:** None

#### Send Risk Report Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email summarizing the generated report.
- **Configuration choices:**  
  - Recipient placeholder:
    - `<__PLACEHOLDER_VALUE__recipient_email__>`
  - Subject:
    - `=Blockchain Risk Analysis Report - {{ $json.output.overallRiskRating }} Risk`
  - Body pulls values from `{{$json.output...}}`
  - Attribution append disabled
- **Key expressions or variables used:**  
  - `{{$json.output.executiveSummary}}`
  - `{{$json.output.smartContractFindings.overallSecurityScore}}`
  - `{{$json.output.smartContractFindings.criticalVulnerabilities.length}}`
  - `{{$json.output.smartContractFindings.highRiskIssues.length}}`
  - `{{$json.output.smartContractFindings.mediumRiskIssues.length}}`
  - `{{$json.output.tokenomicsFindings.daoStabilityScore}}`
  - `{{$json.output.tokenomicsFindings.centralizationRisks.nakamotoCoefficient}}`
  - `{{$json.output.tokenomicsFindings.centralizationRisks.giniCoefficient}}`
  - `{{$json.output.tokenomicsFindings.centralizationRisks.participationRate}}`
  - `{{$json.output.remediationPlan.immediatePriorities.length}}`
  - `{{$json.output.remediationPlan.shortTermActions.length}}`
  - `{{$json.output.remediationPlan.longTermRecommendations.length}}`
  - `{{$json.output.overallRiskRating}}`
- **Input and output connections:**  
  - Input: `Store Risk Analysis`
  - Output: none
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - **Important mapping risk:** this node references `{{$json.output...}}`, but upstream nodes appear to expose fields directly, not necessarily under an `output` object. If the final data is not nested under `output`, the email will fail or produce empty values.
  - Invalid Gmail OAuth2 credentials
  - Gmail send quota or API restrictions
  - Placeholder recipient not replaced
  - Missing nested arrays causing `.length` failures
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Analysis | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Read Solidity Files | ## Trigger, Orchestrate Specialist Agents<br>**What:** Blockchain Risk Orchestrator delegates tasks to Smart Contract Auditor and Tokenomics Sustainability agents.<br>**Why:** Parallel specialised analysis ensures both code-level vulnerabilities and economic risks are assessed simultaneously. |
| Read Solidity Files | n8n-nodes-base.readWriteFile | Read Solidity source file from disk | Start Analysis | Blockchain Risk Orchestrator | ## Trigger, Orchestrate Specialist Agents<br>**What:** Blockchain Risk Orchestrator delegates tasks to Smart Contract Auditor and Tokenomics Sustainability agents.<br>**Why:** Parallel specialised analysis ensures both code-level vulnerabilities and economic risks are assessed simultaneously. |
| Blockchain Risk Orchestrator | @n8n/n8n-nodes-langchain.agent | Main AI coordinator aggregating specialist analyses | Read Solidity Files; Orchestrator Model; Smart Contract Auditor Agent; Tokenomics Sustainability Agent; Structured Risk Report | Prepare Storage Data | ## Trigger, Orchestrate Specialist Agents<br>**What:** Blockchain Risk Orchestrator delegates tasks to Smart Contract Auditor and Tokenomics Sustainability agents.<br>**Why:** Parallel specialised analysis ensures both code-level vulnerabilities and economic risks are assessed simultaneously. |
| Orchestrator Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model used by the orchestration agent |  | Blockchain Risk Orchestrator | ## Setup Steps<br>1. Configure the Read Solidity Files node with the correct disk path to your `.sol` files.<br>2. Add LLM API credentials to all Chat Model nodes (Orchestrator, Auditor, Tokenomics).<br>3. Set static analysis parameters in the Static Analysis Tool node.<br>4. Define risk thresholds and scoring weights in the Risk Score Calculator.<br>5. Configure Monte Carlo simulation parameters (iterations, variables) in the simulation tool node.<br>## Smart Contract Audit<br>**What:** Auditor Agent applies static analysis to detect vulnerabilities in Solidity code.<br>**Why:** Identifies exploitable flaws (e.g., reentrancy, overflow) before deployment. |
| Smart Contract Auditor Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist AI agent for Solidity security review | Auditor Model; Static Analysis Tool; Risk Score Calculator | Blockchain Risk Orchestrator | ## Trigger, Orchestrate Specialist Agents<br>**What:** Blockchain Risk Orchestrator delegates tasks to Smart Contract Auditor and Tokenomics Sustainability agents.<br>**Why:** Parallel specialised analysis ensures both code-level vulnerabilities and economic risks are assessed simultaneously.<br>## Smart Contract Audit<br>**What:** Auditor Agent applies static analysis to detect vulnerabilities in Solidity code.<br>**Why:** Identifies exploitable flaws (e.g., reentrancy, overflow) before deployment. |
| Auditor Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for smart contract analysis |  | Smart Contract Auditor Agent | ## Setup Steps<br>1. Configure the Read Solidity Files node with the correct disk path to your `.sol` files.<br>2. Add LLM API credentials to all Chat Model nodes (Orchestrator, Auditor, Tokenomics).<br>3. Set static analysis parameters in the Static Analysis Tool node.<br>4. Define risk thresholds and scoring weights in the Risk Score Calculator.<br>5. Configure Monte Carlo simulation parameters (iterations, variables) in the simulation tool node.<br>## Smart Contract Audit<br>**What:** Auditor Agent applies static analysis to detect vulnerabilities in Solidity code.<br>**Why:** Identifies exploitable flaws (e.g., reentrancy, overflow) before deployment. |
| Static Analysis Tool | @n8n/n8n-nodes-langchain.toolCode | Regex-based Solidity vulnerability scanner |  | Smart Contract Auditor Agent | ## Setup Steps<br>1. Configure the Read Solidity Files node with the correct disk path to your `.sol` files.<br>2. Add LLM API credentials to all Chat Model nodes (Orchestrator, Auditor, Tokenomics).<br>3. Set static analysis parameters in the Static Analysis Tool node.<br>4. Define risk thresholds and scoring weights in the Risk Score Calculator.<br>5. Configure Monte Carlo simulation parameters (iterations, variables) in the simulation tool node.<br>## Smart Contract Audit<br>**What:** Auditor Agent applies static analysis to detect vulnerabilities in Solidity code.<br>**Why:** Identifies exploitable flaws (e.g., reentrancy, overflow) before deployment. |
| Risk Score Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Shared arithmetic tool for both specialist agents |  | Smart Contract Auditor Agent; Tokenomics Sustainability Agent | ## Smart Contract Audit<br>**What:** Auditor Agent applies static analysis to detect vulnerabilities in Solidity code.<br>**Why:** Identifies exploitable flaws (e.g., reentrancy, overflow) before deployment.<br>## Tokenomics Simulation<br>**What:** Tokenomics Agent runs Monte Carlo simulations and calculates risk scores.<br>**Why:** Models economic sustainability under uncertainty, revealing collapse or inflation scenarios. |
| Tokenomics Sustainability Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist AI agent for tokenomics and DAO stability analysis | Tokenomics Model; Monte Carlo Simulation Tool; Risk Score Calculator | Blockchain Risk Orchestrator | ## Trigger, Orchestrate Specialist Agents<br>**What:** Blockchain Risk Orchestrator delegates tasks to Smart Contract Auditor and Tokenomics Sustainability agents.<br>**Why:** Parallel specialised analysis ensures both code-level vulnerabilities and economic risks are assessed simultaneously.<br>## Tokenomics Simulation<br>**What:** Tokenomics Agent runs Monte Carlo simulations and calculates risk scores.<br>**Why:** Models economic sustainability under uncertainty, revealing collapse or inflation scenarios. |
| Tokenomics Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for tokenomics analysis |  | Tokenomics Sustainability Agent | ## Setup Steps<br>1. Configure the Read Solidity Files node with the correct disk path to your `.sol` files.<br>2. Add LLM API credentials to all Chat Model nodes (Orchestrator, Auditor, Tokenomics).<br>3. Set static analysis parameters in the Static Analysis Tool node.<br>4. Define risk thresholds and scoring weights in the Risk Score Calculator.<br>5. Configure Monte Carlo simulation parameters in the simulation tool node.<br>## Tokenomics Simulation<br>**What:** Tokenomics Agent runs Monte Carlo simulations and calculates risk scores.<br>**Why:** Models economic sustainability under uncertainty, revealing collapse or inflation scenarios. |
| Monte Carlo Simulation Tool | @n8n/n8n-nodes-langchain.toolCode | Simulation engine for supply, inflation, and governance centralization |  | Tokenomics Sustainability Agent | ## Setup Steps<br>1. Configure the Read Solidity Files node with the correct disk path to your `.sol` files.<br>2. Add LLM API credentials to all Chat Model nodes (Orchestrator, Auditor, Tokenomics).<br>3. Set static analysis parameters in the Static Analysis Tool node.<br>4. Define risk thresholds and scoring weights in the Risk Score Calculator.<br>5. Configure Monte Carlo simulation parameters in the simulation tool node.<br>## Tokenomics Simulation<br>**What:** Tokenomics Agent runs Monte Carlo simulations and calculates risk scores.<br>**Why:** Models economic sustainability under uncertainty, revealing collapse or inflation scenarios. |
| Structured Risk Report | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces final structured JSON report schema |  | Blockchain Risk Orchestrator | ## Tokenomics Simulation<br>**What:** Tokenomics Agent runs Monte Carlo simulations and calculates risk scores.<br>**Why:** Models economic sustainability under uncertainty, revealing collapse or inflation scenarios.<br>## Parse, Store & Report<br>**What:** Structured Risk Report Parser consolidates findings; data is stored and emailed via Gmail.<br>**Why:** Delivers a traceable, shareable audit trail to all stakeholders automatically. |
| Prepare Storage Data | n8n-nodes-base.set | Add timestamp and storage-oriented summary fields | Blockchain Risk Orchestrator | Store Risk Analysis | ## Parse, Store & Report<br>**What:** Structured Risk Report Parser consolidates findings; data is stored and emailed via Gmail.<br>**Why:** Delivers a traceable, shareable audit trail to all stakeholders automatically. |
| Store Risk Analysis | n8n-nodes-base.dataTable | Persist analysis into n8n Data Table | Prepare Storage Data | Send Risk Report Email | ## Parse, Store & Report<br>**What:** Structured Risk Report Parser consolidates findings; data is stored and emailed via Gmail.<br>**Why:** Delivers a traceable, shareable audit trail to all stakeholders automatically. |
| Send Risk Report Email | n8n-nodes-base.gmail | Send report summary to stakeholders via Gmail | Store Risk Analysis |  | ## Parse, Store & Report<br>**What:** Structured Risk Report Parser consolidates findings; data is stored and emailed via Gmail.<br>**Why:** Delivers a traceable, shareable audit trail to all stakeholders automatically. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas setup note |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas explanatory note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas setup note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas block description note |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas block description note |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas block description note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a step-by-step reconstruction plan.

## 4.1 Create the entry and file input

1. **Create a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Name it: **Start Analysis**

2. **Create a Read/Write Files from Disk node**
   - Node type: **Read/Write Files from Disk**
   - Name it: **Read Solidity Files**
   - Operation: read file
   - Set the file path in **File Selector** to your Solidity file location, for example:
     - `/path/to/contract.sol`
   - Leave extra options empty unless your environment requires them

3. **Connect**
   - `Start Analysis` → `Read Solidity Files`

## 4.2 Create the orchestrator AI layer

4. **Create an AI Agent node**
   - Node type: **LangChain Agent**
   - Name it: **Blockchain Risk Orchestrator**
   - Set **Prompt Type** to **Define**
   - Set **Text** to:
     - `={{ $json.data }}`
   - Add this **System Message**:
     - “You are a Blockchain Risk Orchestrator coordinating specialized agents to analyze smart contracts and tokenomics. You delegate smart contract static analysis to the Smart Contract Auditor Agent and tokenomics sustainability analysis to the Tokenomics Sustainability Agent. Aggregate their findings into a comprehensive risk assessment with actionable remediation plans.”
   - Enable **Output Parser**

5. **Create a Chat Model node for the orchestrator**
   - Node type: **OpenAI Chat Model**
   - Name it: **Orchestrator Model**
   - Model: **gpt-4o**
   - Temperature: **0.2**
   - Credentials: configure your **OpenAI API** credential

6. **Connect**
   - `Orchestrator Model` → `Blockchain Risk Orchestrator` using the **AI language model** connection

## 4.3 Create the structured output parser

7. **Create a Structured Output Parser node**
   - Node type: **Structured Output Parser**
   - Name it: **Structured Risk Report**
   - Schema type: **Manual**
   - Paste a schema defining these top-level required fields:
     - `executiveSummary`
     - `smartContractFindings`
     - `tokenomicsFindings`
     - `remediationPlan`
     - `overallRiskRating`

8. **Define nested schema contents**
   - `smartContractFindings`:
     - `criticalVulnerabilities` array of objects
     - `highRiskIssues` array of objects
     - `mediumRiskIssues` array of objects
     - `overallSecurityScore` number
   - `tokenomicsFindings`:
     - `daoStabilityScore` number
     - `centralizationRisks` object with:
       - `nakamotoCoefficient`
       - `giniCoefficient`
       - `whaleVotingPower`
       - `participationRate`
       - `riskLevel`
     - `inflationAnalysis` object
     - `supplyDynamics` object
   - `remediationPlan`:
     - `immediatePriorities` array
     - `shortTermActions` array
     - `longTermRecommendations` array
   - `overallRiskRating` as string

9. **Connect**
   - `Structured Risk Report` → `Blockchain Risk Orchestrator` using the **AI output parser** connection

## 4.4 Create the smart contract specialist branch

10. **Create an AI Agent Tool node**
    - Node type: **AI Agent Tool**
    - Name it: **Smart Contract Auditor Agent**
    - Set **Text** to:
      - `={{ $fromAI('solidity_code', 'The Solidity smart contract source code to analyze', 'string') }}`
    - Set **Tool Description** to indicate it performs Solidity static analysis
    - Use this **System Message**:
      - “You are a Smart Contract Security Auditor specializing in Solidity static analysis. Analyze smart contract code for: 1) Reentrancy vulnerabilities (check-effects-interactions pattern violations), 2) Integer overflow/underflow risks (pre-0.8.0 Solidity), 3) Unchecked external calls and return values, 4) Access control flaws (missing modifiers, improper visibility), 5) Logic errors and edge cases. Use the static analysis tool to perform pattern matching and the calculator for risk scoring. Provide severity ratings (Critical/High/Medium/Low) and specific line references.”

11. **Create a Chat Model node**
    - Node type: **OpenAI Chat Model**
    - Name it: **Auditor Model**
    - Model: **gpt-4o**
    - Temperature: **0.1**
    - Credentials: same OpenAI credential or equivalent

12. **Connect**
    - `Auditor Model` → `Smart Contract Auditor Agent` using **AI language model**

13. **Create a Code Tool node**
    - Node type: **AI Tool Code**
    - Name it: **Static Analysis Tool**
    - Paste JavaScript logic that:
      - reads `code` and optionally `analysisType`
      - splits the Solidity source into lines
      - scans for:
        - external call patterns such as `.call{}`, `.transfer()`, `.send()`
        - state changes after external calls
        - unchecked `.call{}`
        - arithmetic risks in Solidity `<0.8.0`
        - public sensitive functions lacking basic access control
      - returns:
        - `findings`
        - `totalIssues`
        - `analysisComplete: true`

14. **Connect**
    - `Static Analysis Tool` → `Smart Contract Auditor Agent` using **AI tool**

15. **Create a Calculator Tool node**
    - Node type: **AI Tool Calculator**
    - Name it: **Risk Score Calculator**

16. **Connect**
    - `Risk Score Calculator` → `Smart Contract Auditor Agent` using **AI tool**

17. **Connect the smart contract specialist to the orchestrator**
    - `Smart Contract Auditor Agent` → `Blockchain Risk Orchestrator` using **AI tool**

## 4.5 Create the tokenomics specialist branch

18. **Create an AI Agent Tool node**
    - Node type: **AI Agent Tool**
    - Name it: **Tokenomics Sustainability Agent**
    - Set **Text** to:
      - `={{ $fromAI('tokenomics_data', 'Token economics parameters including supply, distribution, governance rules, and emission schedules', 'string') }}`
    - Set **Tool Description** to indicate Monte Carlo and governance sustainability analysis
    - Use this **System Message**:
      - “You are a Tokenomics Sustainability Analyst specializing in DAO governance risk assessment. Analyze token economics for: 1) Token supply dynamics and inflation scenarios using Monte Carlo simulations, 2) Governance centralization risks (voting power concentration, whale dominance), 3) Economic sustainability (token velocity, liquidity depth, emission schedules), 4) DAO stability metrics (Gini coefficient, Nakamoto coefficient, governance participation rates). Use the Monte Carlo simulation tool to model probabilistic scenarios and the calculator for statistical analysis. Provide a DAO stability score (0-100) and identify systemic risks.”

19. **Create a Chat Model node**
    - Node type: **OpenAI Chat Model**
    - Name it: **Tokenomics Model**
    - Model: **gpt-4o**
    - Temperature: **0.1**
    - Credentials: OpenAI API credential

20. **Connect**
    - `Tokenomics Model` → `Tokenomics Sustainability Agent` using **AI language model**

21. **Create a Code Tool node**
    - Node type: **AI Tool Code**
    - Name it: **Monte Carlo Simulation Tool**
    - Paste JavaScript logic that:
      - reads parameters from the input JSON
      - supports defaults for:
        - `simulations`
        - `timeHorizon`
        - `initialSupply`
        - `inflationRate`
        - `distributionGini`
        - `topHoldersPercent`
      - simulates supply growth
      - simulates governance centralization and participation
      - computes:
        - means
        - standard deviations
        - percentiles
        - average Nakamoto coefficient
        - average participation rate
        - DAO stability score
      - returns a JSON object containing:
        - `supplyProjections`
        - `inflationScenarios`
        - `centralizationRisks`
        - `stabilityMetrics`

22. **Connect**
    - `Monte Carlo Simulation Tool` → `Tokenomics Sustainability Agent` using **AI tool**

23. **Reuse the same Calculator Tool**
    - Connect `Risk Score Calculator` → `Tokenomics Sustainability Agent` using **AI tool**

24. **Connect the tokenomics specialist to the orchestrator**
    - `Tokenomics Sustainability Agent` → `Blockchain Risk Orchestrator` using **AI tool**

## 4.6 Connect the main execution path

25. **Connect file input to orchestrator**
    - `Read Solidity Files` → `Blockchain Risk Orchestrator`

26. **Create a Set node**
    - Node type: **Set**
    - Name it: **Prepare Storage Data**
    - Enable **Include Other Fields**
    - Add these fields:
      1. `timestamp` as string:
         - `={{ $now.toISO() }}`
      2. `overallRiskRating` as string:
         - `={{ $json.overallRiskRating }}`
      3. `securityScore` as number:
         - `={{ $json.smartContractFindings.overallSecurityScore }}`
      4. `daoStabilityScore` as number:
         - `={{ $json.tokenomicsFindings.daoStabilityScore }}`
      5. `criticalVulnerabilities` as number:
         - `={{ $json.smartContractFindings.criticalVulnerabilities.length }}`
      6. `executiveSummary` as string:
         - `={{ $json.executiveSummary }}`

27. **Connect**
    - `Blockchain Risk Orchestrator` → `Prepare Storage Data`

## 4.7 Configure storage

28. **Create a Data Table node**
    - Node type: **Data Table**
    - Name it: **Store Risk Analysis**
    - Choose your target table
    - Set **Mapping Mode** to **Auto-map input data**
    - Ensure the table can accept all fields you want to store, especially:
      - timestamp
      - overallRiskRating
      - securityScore
      - daoStabilityScore
      - criticalVulnerabilities
      - executiveSummary
      - and optionally the full structured report fields

29. **Connect**
    - `Prepare Storage Data` → `Store Risk Analysis`

## 4.8 Configure Gmail notification

30. **Create a Gmail node**
    - Node type: **Gmail**
    - Name it: **Send Risk Report Email**
    - Operation: **Send Email**
    - Configure **Gmail OAuth2** credentials
    - Set recipient to your desired email address
    - Subject:
      - `Blockchain Risk Analysis Report - {{ $json.output.overallRiskRating }} Risk`
    - Message body with sections for:
      - executive summary
      - smart contract findings
      - tokenomics findings
      - remediation plan counts
      - overall risk rating

31. **Important correction recommended**
    - Because the workflow currently stores parsed fields directly at top level, replace `{{$json.output...}}` with `{{$json...}}` unless your actual agent output is nested under `output`.
    - Safer subject:
      - `Blockchain Risk Analysis Report - {{ $json.overallRiskRating }} Risk`
    - Safer body examples:
      - `{{ $json.executiveSummary }}`
      - `{{ $json.smartContractFindings.overallSecurityScore }}`
      - `{{ $json.tokenomicsFindings.daoStabilityScore }}`

32. **Connect**
    - `Store Risk Analysis` → `Send Risk Report Email`

## 4.9 Credentials to configure

33. **OpenAI credentials**
    - Required for:
      - Orchestrator Model
      - Auditor Model
      - Tokenomics Model
    - Use an account with access to `gpt-4o`

34. **Gmail OAuth2 credentials**
    - Required for:
      - Send Risk Report Email
    - Confirm Gmail API access and sender permissions

35. **Filesystem access**
    - Required for:
      - Read Solidity Files
    - The n8n runtime must be able to access the specified path

36. **Data Table setup**
    - Required for:
      - Store Risk Analysis
    - Create or select a table before execution

## 4.10 Recommended hardening before production use

37. **Add tokenomics input source**
    - The current workflow only reads Solidity code
    - Add a separate input source for tokenomics parameters such as:
      - JSON file
      - form submission
      - database record
      - webhook payload

38. **Validate parser outputs before storage/email**
    - Add an IF node or Code node to verify required fields exist

39. **Normalize email expressions**
    - Use top-level fields consistently

40. **Reduce simulation cost if needed**
    - Lower `simulations` from `10000` if execution time is too high

41. **Improve static analysis accuracy**
    - Replace regex-only logic with a real parser or external analyzer if higher confidence is required

42. **Consider replacing Manual Trigger for automation**
    - Possible alternatives:
      - Cron
      - Webhook
      - Email trigger
      - Git-based event source

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or external sub-workflow nodes.  
The specialist agents are internal AI tool-agents, not separate n8n workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates blockchain risk assessment using a multi-agent AI architecture, targeting DeFi developers, blockchain auditors, and Web3 project teams who need rigorous smart contract and tokenomics evaluation before deployment or investment decisions. | From canvas note |
| The workflow aims to reduce the time-intensive, expertise-heavy process of manually auditing Solidity contracts and modeling tokenomics sustainability. | From canvas note |
| Setup guidance: configure the Solidity file path, add LLM credentials to all three Chat Model nodes, tune static analysis parameters, define calculator scoring weights, and configure Monte Carlo simulation parameters. | From canvas notes |
| Functional flow note: the Blockchain Risk Orchestrator delegates tasks to Smart Contract Auditor and Tokenomics Sustainability agents so code-level and economic risks can be analyzed in parallel. | From canvas note |
| Smart contract note: the Auditor Agent applies static analysis to detect exploitable flaws such as reentrancy and overflow before deployment. | From canvas note |
| Tokenomics note: the Tokenomics Agent runs Monte Carlo simulations and calculates risk scores to model sustainability under uncertainty. | From canvas note |
| Reporting note: the Structured Risk Report parser consolidates findings; data is stored and emailed via Gmail for a traceable audit trail. | From canvas note |

## Additional implementation cautions

- The workflow title mentions **Gmail**, and the workflow indeed uses Gmail.
- The title mentions **GPT-4o**, and all three language model nodes use `gpt-4o`.
- The title mentions **tokenomics risk**, but the actual workflow input only includes Solidity file content. Tokenomics analysis therefore lacks a dedicated factual input source unless the orchestrator infers it.
- The internal workflow name includes **“monte carlo”**, which matches the custom simulation tool.
- The email node likely needs expression correction from `output.*` to direct fields unless runtime testing confirms a nested structure.