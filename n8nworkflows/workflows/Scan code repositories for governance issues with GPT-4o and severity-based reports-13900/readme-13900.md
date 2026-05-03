Scan code repositories for governance issues with GPT-4o and severity-based reports

https://n8nworkflows.xyz/workflows/scan-code-repositories-for-governance-issues-with-gpt-4o-and-severity-based-reports-13900


# Scan code repositories for governance issues with GPT-4o and severity-based reports

# 1. Workflow Overview

This workflow performs an AI-driven governance scan of a source code repository and produces a structured compliance report. It combines repository file discovery, multi-agent LLM analysis, structured report generation, and severity-based routing for escalation or standard logging.

Primary use cases:
- Automated code governance reviews before release
- Technical debt and architecture compliance assessment
- Security-oriented repository scanning
- Executive reporting for engineering leadership and CTO-level stakeholders

The workflow is organized into these logical blocks:

## 1.1 Entry and Repository Discovery
The workflow starts manually, then uses an SSH command to list code files from a repository path. This file listing becomes the raw input for downstream AI analysis.

## 1.2 Multi-Agent Governance Analysis
A central orchestration agent receives repository data and delegates work to specialized AI sub-agents:
- Static code analysis
- Architecture compliance analysis
- Security vulnerability scanning
- CTO/executive report synthesis

A structured output parser forces the orchestrator’s final response into a defined governance report schema.

## 1.3 Report Formatting
The structured AI result is wrapped with timestamps and repository context in a formatting node.

## 1.4 Critical Threshold Check and Severity Routing
The workflow checks whether the number of critical issues exceeds a threshold. It then either prepares an escalation payload or logs a standard report, and routes findings by overall health score into severity handlers.

## 1.5 Aggregation, Merge, and Final Enrichment
Critical and medium paths are aggregated, merged with standard report output, and enriched with final workflow metadata for delivery or downstream consumption.

---

# 2. Block-by-Block Analysis

## 2.1 Entry and Repository Discovery

### Overview
This block initializes the workflow and collects a list of repository files to scan. It is the only execution entry point in the workflow.

### Nodes Involved
- Start Governance Scan
- Extract Repository Metadata

### Node Details

#### Start Governance Scan
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual execution entry point.
- **Configuration choices:** No parameters are configured. It starts the workflow on demand from the n8n editor.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Extract Repository Metadata**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime beyond standard manual execution availability.
- **Sub-workflow reference:** None.

#### Extract Repository Metadata
- **Type and technical role:** `n8n-nodes-base.ssh`; executes a shell command remotely or through configured SSH credentials to enumerate repository files.
- **Configuration choices:** Runs a `find` command against `{{$json.repositoryPath}}`, filtering for common source code extensions:
  - `.js`
  - `.ts`
  - `.py`
  - `.java`
  - `.go`
  It limits output to the first 100 matching files with `head -100`.
- **Key expressions or variables used:**
  - `{{ $json.repositoryPath }}`
- **Input and output connections:** Input from **Start Governance Scan**; output to **Governance Orchestrator Agent**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Missing `repositoryPath` in incoming JSON
  - Invalid filesystem path
  - SSH authentication or host connectivity failure
  - Permission denied on the target path
  - Empty result set if no matching files exist
  - The command only returns file paths, not file contents
- **Sub-workflow reference:** None.

---

## 2.2 Multi-Agent Governance Analysis

### Overview
This is the core AI analysis block. A LangChain-based orchestration agent receives repository data, calls specialized analysis tools, and returns a structured governance report aligned with a strict schema.

### Nodes Involved
- Governance Orchestrator Agent
- Orchestrator Model
- Static Code Analysis Agent
- Static Analysis Model
- Architectural Compliance Agent
- Architectural Analysis Model
- CTO Report Generation Agent
- Report Generation Model
- Security Vulnerability Scanner Agent
- Security Analysis Model
- Structured Governance Output

### Node Details

#### Governance Orchestrator Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; central AI coordinator.
- **Configuration choices:**
  - Input text is `{{ $json.codebaseData }}`
  - Uses a system message defining its role as a software governance orchestrator
  - Has output parser enabled
- **Key expressions or variables used:**
  - `{{ $json.codebaseData }}`
- **Input and output connections:**
  - Main input from **Extract Repository Metadata**
  - AI language model input from **Orchestrator Model**
  - AI tool connections from:
    - **Static Code Analysis Agent**
    - **Architectural Compliance Agent**
    - **CTO Report Generation Agent**
    - **Security Vulnerability Scanner Agent**
  - AI output parser from **Structured Governance Output**
  - Main output to **Format Final Report**
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain agent support in n8n.
- **Edge cases or potential failure types:**
  - `codebaseData` is not actually produced upstream in this workflow
  - Upstream SSH node returns `stdout`, not `codebaseData`
  - Tool invocation may fail if the agent does not supply required tool variables
  - Token/context overflow if repository data becomes too large
  - OpenAI credential or API quota issues
  - Structured parsing failure if response does not match schema
- **Sub-workflow reference:** None.

#### Orchestrator Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for the orchestrator.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - No built-in tools configured
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to **Governance Orchestrator Agent**.
- **Version-specific requirements:** Type version `1.3`; requires OpenAI credentials.
- **Edge cases or potential failure types:**
  - Invalid or expired OpenAI API credentials
  - Model availability mismatch
  - Rate limiting or token quota exhaustion
- **Sub-workflow reference:** None.

#### Static Code Analysis Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; a tool callable by the orchestrator for code quality analysis.
- **Configuration choices:**
  - Tool input comes from AI variable:
    `{{ $fromAI("codeFiles", "The code files and repository structure to analyze", "string") }}`
  - System message instructs the tool to evaluate SOLID violations, code smells, anti-patterns, maintainability risks, and compute a technical debt index.
  - Output parser enabled.
  - Tool description is provided for the orchestrator.
- **Key expressions or variables used:**
  - `$fromAI("codeFiles", ...)`
- **Input and output connections:**
  - AI language model from **Static Analysis Model**
  - AI tool output to **Governance Orchestrator Agent**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Tool may receive only file paths, not code contents
  - AI may fabricate line numbers if actual content is unavailable
  - Parser mismatch if output format is inconsistent
  - Large codebase context may exceed limits
- **Sub-workflow reference:** None.

#### Static Analysis Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model powering the static analysis tool.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to **Static Code Analysis Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Standard OpenAI auth, quota, timeout, or model availability failures.
- **Sub-workflow reference:** None.

#### Architectural Compliance Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; architecture validation tool callable by the orchestrator.
- **Configuration choices:**
  - Tool input:
    `{{ $fromAI("architectureData", "The repository structure and architectural metadata to validate", "string") }}`
  - System message focuses on:
    - service boundaries
    - API contracts
    - data ownership
    - coupling
    - dependency graph stability
    - scalability constraints
    - resilience patterns
  - Output parser enabled.
- **Key expressions or variables used:**
  - `$fromAI("architectureData", ...)`
- **Input and output connections:**
  - AI language model from **Architectural Analysis Model**
  - AI tool output to **Governance Orchestrator Agent**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Architectural metadata is not explicitly generated upstream
  - Limited evidence if only file paths are provided
  - Hallucinated architectural assumptions are possible
- **Sub-workflow reference:** None.

#### Architectural Analysis Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for architectural compliance analysis.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to **Architectural Compliance Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Standard OpenAI-related issues.
- **Sub-workflow reference:** None.

#### CTO Report Generation Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; executive synthesis tool.
- **Configuration choices:**
  - Tool input:
    `{{ $fromAI("analysisResults", "The combined technical analysis results from code and architecture agents", "string") }}`
  - System message requests:
    - risk matrix
    - remediation backlog prioritization
    - compliance summary
    - ROI/resource/timeline framing
  - Output parser enabled.
- **Key expressions or variables used:**
  - `$fromAI("analysisResults", ...)`
- **Input and output connections:**
  - AI language model from **Report Generation Model**
  - AI tool output to **Governance Orchestrator Agent**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Combined analysis results must be assembled by the orchestrator
  - If upstream tools produce weak or partial output, executive summary quality degrades
- **Sub-workflow reference:** None.

#### Report Generation Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for executive report generation.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to **CTO Report Generation Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Standard OpenAI-related issues.
- **Sub-workflow reference:** None.

#### Security Vulnerability Scanner Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; security-focused analysis tool callable by the orchestrator.
- **Configuration choices:**
  - Tool input:
    `{{ $fromAI('securityContext', 'The code and dependency information to scan for security vulnerabilities', 'string') }}`
  - System message targets:
    - injection vulnerabilities
    - auth flaws
    - secrets exposure
    - insecure dependencies
    - misconfigurations
    - cryptographic failures
    - OWASP Top 10 mapping
    - CVSS/CVE references
  - Tool description provided
  - No explicit output parser enabled in parameters
- **Key expressions or variables used:**
  - `$fromAI('securityContext', ...)`
- **Input and output connections:**
  - AI language model from **Security Analysis Model**
  - AI tool output to **Governance Orchestrator Agent**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Dependency metadata is not actually collected upstream
  - CVE mapping may be speculative without package manifests or versions
  - File-path-only input weakens security analysis quality
- **Sub-workflow reference:** None.

#### Security Analysis Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for the security tool.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to **Security Vulnerability Scanner Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Standard OpenAI-related issues.
- **Sub-workflow reference:** None.

#### Structured Governance Output
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces JSON-schema-like structured output from the orchestrator.
- **Configuration choices:**
  - Manual schema with these top-level fields:
    - `riskMatrix[]`
    - `remediationBacklog[]`
    - `complianceSummary`
    - `executiveSummary`
    - `recommendations[]`
  - `complianceSummary` requires:
    - `technicalDebtIndex`
    - `architectureComplianceScore`
    - `overallHealthScore`
    - `criticalIssuesCount`
    - `highPriorityCount`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI output parser connection to **Governance Orchestrator Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - LLM output may fail schema validation
  - Missing required numeric fields can break downstream severity logic
- **Sub-workflow reference:** None.

---

## 2.3 Report Formatting

### Overview
This block adds metadata to the AI-generated governance output. It prepares the payload for threshold checks and downstream routing.

### Nodes Involved
- Format Final Report

### Node Details

#### Format Final Report
- **Type and technical role:** `n8n-nodes-base.set`; formats and enriches the orchestrator output.
- **Configuration choices:**
  - Adds `reportGeneratedAt` with current ISO timestamp
  - Adds `repositoryScanned` from SSH stdout
  - Adds `governanceReport` as `JSON.stringify($json)`
  - Keeps existing fields
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $('Extract Repository Metadata').item.json.stdout }}`
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:** Input from **Governance Orchestrator Agent**; output to **Check Critical Issues Threshold**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `governanceReport` is stored as a string even though its type is declared as object
  - Downstream nodes incorrectly access nested properties inside `governanceReport` as if it were a real object
  - Large JSON string may make debugging harder
- **Sub-workflow reference:** None.

---

## 2.4 Critical Threshold Check and Severity Routing

### Overview
This block decides whether the report requires escalation and then assigns severity-specific handling based on health score. It contains the main decision logic of the workflow.

### Nodes Involved
- Check Critical Issues Threshold
- Prepare Escalation Alert
- Route by Severity Level
- Critical Severity Handler
- Medium Severity Handler
- Log Standard Report

### Node Details

#### Check Critical Issues Threshold
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the number of critical issues exceeds a threshold.
- **Configuration choices:**
  - Condition:
    `{{ $('Format Final Report').item.json.governanceReport.complianceSummary.criticalIssuesCount }} > 5`
- **Key expressions or variables used:**
  - `{{ $('Format Final Report').item.json.governanceReport.complianceSummary.criticalIssuesCount }}`
- **Input and output connections:**
  - Input from **Format Final Report**
  - True output to **Prepare Escalation Alert**
  - False output to **Log Standard Report**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - This expression is likely broken because `governanceReport` was stringified, so `governanceReport.complianceSummary` will not exist as an object
  - If parser output omitted `complianceSummary`, condition fails
  - Loose type validation may mask malformed values
- **Sub-workflow reference:** None.

#### Prepare Escalation Alert
- **Type and technical role:** `n8n-nodes-base.set`; prepares an escalation payload when critical issue count exceeds threshold.
- **Configuration choices:**
  - Sets:
    - `alertType = CRITICAL_THRESHOLD_EXCEEDED`
    - `criticalIssuesCount = {{ $json.complianceSummary.criticalIssuesCount }}`
    - `escalationRequired = true`
    - `alertTimestamp = {{ $now.toISO() }}`
    - `fullReport = {{ JSON.stringify($json) }}`
- **Key expressions or variables used:**
  - `{{ $json.complianceSummary.criticalIssuesCount }}`
  - `{{ $now.toISO() }}`
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:** Input from **Check Critical Issues Threshold** true branch; output to **Route by Severity Level**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Assumes `complianceSummary` is still accessible on current item
  - `fullReport` also becomes a stringified payload
- **Sub-workflow reference:** None.

#### Route by Severity Level
- **Type and technical role:** `n8n-nodes-base.switch`; routes reports according to overall health score.
- **Configuration choices:**
  - Rule 1: if `overallHealthScore < 30`, route to output 0
  - Rule 2: if `overallHealthScore < 70`, route to output 1
  - No explicit fallback/default branch connected
- **Key expressions or variables used:**
  - `{{ $json.complianceSummary.overallHealthScore }}`
- **Input and output connections:**
  - Input from **Prepare Escalation Alert**
  - Output 0 to **Critical Severity Handler**
  - Output 1 to **Medium Severity Handler**
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**
  - Scores under 30 also satisfy under 70; switch rule behavior matters, but n8n routes by ordered rule/output logic
  - Reports with score `>= 70` are dropped because no connected fallback exists
  - This routing only runs for escalation path, not standard reports
- **Sub-workflow reference:** None.

#### Critical Severity Handler
- **Type and technical role:** `n8n-nodes-base.set`; tags an item as critical.
- **Configuration choices:**
  - Sets:
    - `severityLevel = CRITICAL`
    - `actionRequired = IMMEDIATE_REMEDIATION`
    - `notificationPriority = P1`
    - `escalationPath = CTO_DIRECT`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Route by Severity Level**; output to **Aggregate Critical Findings**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Minimal; mostly dependent on upstream routing.
- **Sub-workflow reference:** None.

#### Medium Severity Handler
- **Type and technical role:** `n8n-nodes-base.set`; tags an item as medium severity.
- **Configuration choices:**
  - Sets:
    - `severityLevel = MEDIUM`
    - `actionRequired = SCHEDULED_REMEDIATION`
    - `notificationPriority = P2`
    - `escalationPath = ENGINEERING_LEAD`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Route by Severity Level**; output to **Aggregate Critical Findings**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Minimal.
- **Sub-workflow reference:** None.

#### Log Standard Report
- **Type and technical role:** `n8n-nodes-base.set`; records metadata for non-escalated reports.
- **Configuration choices:**
  - Sets:
    - `reportStatus = STANDARD_COMPLIANCE`
    - `loggedAt = {{ $now.toISO() }}`
    - `summary = {{ $json.executiveSummary }}`
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $json.executiveSummary }}`
- **Input and output connections:** Input from **Check Critical Issues Threshold** false branch; output to **Merge Analysis Paths**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Assumes `executiveSummary` exists
  - This branch bypasses severity routing entirely
- **Sub-workflow reference:** None.

---

## 2.5 Aggregation, Merge, and Final Enrichment

### Overview
This block consolidates routed outputs and appends final workflow metadata. It is the final normalization stage before delivery.

### Nodes Involved
- Aggregate Critical Findings
- Merge Analysis Paths
- Enrich Final Output

### Node Details

#### Aggregate Critical Findings
- **Type and technical role:** `n8n-nodes-base.aggregate`; aggregates all incoming severity-handled items.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Critical Severity Handler** and **Medium Severity Handler**; output to **Merge Analysis Paths**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Despite the name, it aggregates both critical and medium findings
  - If only one item arrives, aggregation still wraps/changes shape
  - Downstream shape may differ from standard report branch shape
- **Sub-workflow reference:** None.

#### Merge Analysis Paths
- **Type and technical role:** `n8n-nodes-base.merge`; merges escalation-side and standard-report-side outputs.
- **Configuration choices:** Default merge behavior; no explicit mode is shown.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input 0 from **Log Standard Report**
  - Input 1 from **Aggregate Critical Findings**
  - Output to **Enrich Final Output**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Merge behavior depends on item counts and timing
  - Because branches are mutually exclusive after the IF node, one side may be empty
  - Default merge settings may not behave as expected for asynchronous one-sided inputs
- **Sub-workflow reference:** None.

#### Enrich Final Output
- **Type and technical role:** `n8n-nodes-base.set`; appends completion metadata to the final result.
- **Configuration choices:**
  - Sets:
    - `workflowCompletedAt = {{ $now.toISO() }}`
    - `totalAgentsInvolved = 5`
    - `analysisDepth = COMPREHENSIVE_MULTI_AGENT`
    - `frameworkVersion = 2.0-ENHANCED`
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
- **Input and output connections:** Input from **Merge Analysis Paths**; no downstream node connected.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `totalAgentsInvolved` counts four agent tools plus orchestrator
  - No explicit delivery node exists after this step
- **Sub-workflow reference:** None.

---

## 2.6 Documentation and In-Canvas Notes

### Overview
These nodes are non-executable annotations used to describe prerequisites, setup, severity logic, and workflow purpose.

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
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Contains prerequisites, use cases, customization guidance, and benefits.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; setup guidance.
- **Configuration choices:** Documents setup steps for repository metadata extraction, severity threshold tuning, and escalation configuration.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; high-level process explanation.
- **Configuration choices:** Describes the end-to-end orchestration pattern and intended stakeholders.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; severity routing explanation.
- **Configuration choices:** Documents purpose of routing findings to severity handlers.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; formatting explanation.
- **Configuration choices:** Describes report consolidation before severity assessment.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explanation for extraction/orchestration/sub-agent zone.
- **Configuration choices:** Describes decomposition into specialized agents.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explanation for final aggregation zone.
- **Configuration choices:** Documents aggregation, merge, and delivery purpose.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Governance Scan | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Extract Repository Metadata | ## How It Works<br>This workflow automates end-to-end code repository governance scanning using a multi-agent AI orchestration system. Designed for engineering leads, DevSecOps teams, and CTOs, it replaces manual code audits with a structured, AI-driven compliance and security analysis pipeline. The workflow begins by extracting repository metadata, which is passed to a Governance Orchestrator Agent coordinating four specialised sub-agents: Static Code Analysis, Architectural Compliance, CTO Report Generation, and Security Vulnerability Analysis. Outputs are consolidated into a Structured Governance Output, formatted as a final report, then routed by severity level. Critical findings trigger escalation alerts and are aggregated separately, while medium findings are handled independently. All paths converge to merge analysis results, enrich the final output, and deliver a board-ready governance report with full audit traceability.<br>## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Extract Repository Metadata | n8n-nodes-base.ssh | Repository file discovery over SSH | Start Governance Scan | Governance Orchestrator Agent | ## Setup Steps<br>1. Configure `Extract Repository Metadata` with your Git provider or repository API credentials.<br>2. Set severity thresholds in the `Check Critical Issues Threshold` node to match your governance policy.<br>3. Configure `Prepare Escalation Alert` with your notification channel.<br>## How It Works<br>This workflow automates end-to-end code repository governance scanning using a multi-agent AI orchestration system. Designed for engineering leads, DevSecOps teams, and CTOs, it replaces manual code audits with a structured, AI-driven compliance and security analysis pipeline. The workflow begins by extracting repository metadata, which is passed to a Governance Orchestrator Agent coordinating four specialised sub-agents: Static Code Analysis, Architectural Compliance, CTO Report Generation, and Security Vulnerability Analysis. Outputs are consolidated into a Structured Governance Output, formatted as a final report, then routed by severity level. Critical findings trigger escalation alerts and are aggregated separately, while medium findings are handled independently. All paths converge to merge analysis results, enrich the final output, and deliver a board-ready governance report with full audit traceability.<br>## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Governance Orchestrator Agent | @n8n/n8n-nodes-langchain.agent | Central AI coordinator for all analysis tools | Extract Repository Metadata; Orchestrator Model; Static Code Analysis Agent; Architectural Compliance Agent; CTO Report Generation Agent; Security Vulnerability Scanner Agent; Structured Governance Output | Format Final Report | ## How It Works<br>This workflow automates end-to-end code repository governance scanning using a multi-agent AI orchestration system. Designed for engineering leads, DevSecOps teams, and CTOs, it replaces manual code audits with a structured, AI-driven compliance and security analysis pipeline. The workflow begins by extracting repository metadata, which is passed to a Governance Orchestrator Agent coordinating four specialised sub-agents: Static Code Analysis, Architectural Compliance, CTO Report Generation, and Security Vulnerability Analysis. Outputs are consolidated into a Structured Governance Output, formatted as a final report, then routed by severity level. Critical findings trigger escalation alerts and are aggregated separately, while medium findings are handled independently. All paths converge to merge analysis results, enrich the final output, and deliver a board-ready governance report with full audit traceability.<br>## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Orchestrator Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for orchestrator |  | Governance Orchestrator Agent | ## Prerequisites<br>- OpenAI or compatible LLM API credentials<br>- Git repository access (GitHub, GitLab, or Bitbucket API)<br>- Notification channel (Slack, email, or webhook)<br>## Use Cases<br>- Automated pre-release security and compliance audits<br>## Customisation<br>- Adjust severity thresholds to match internal risk frameworks<br>## Benefits<br>- Eliminates manual code audit effort across engineering teams |
| Static Code Analysis Agent | @n8n/n8n-nodes-langchain.agentTool | Tool for static code quality and technical debt analysis | Static Analysis Model | Governance Orchestrator Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Static Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for static analysis tool |  | Static Code Analysis Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Architectural Compliance Agent | @n8n/n8n-nodes-langchain.agentTool | Tool for architecture and microservices compliance analysis | Architectural Analysis Model | Governance Orchestrator Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Architectural Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for architecture analysis tool |  | Architectural Compliance Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| CTO Report Generation Agent | @n8n/n8n-nodes-langchain.agentTool | Tool for executive-level governance synthesis | Report Generation Model | Governance Orchestrator Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Report Generation Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for CTO report synthesis |  | CTO Report Generation Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Security Vulnerability Scanner Agent | @n8n/n8n-nodes-langchain.agentTool | Tool for security vulnerability scanning | Security Analysis Model | Governance Orchestrator Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Security Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for security analysis tool |  | Security Vulnerability Scanner Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Structured Governance Output | @n8n/n8n-nodes-langchain.outputParserStructured | Schema-enforced structured AI output |  | Governance Orchestrator Agent | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Format Final Report | n8n-nodes-base.set | Adds report metadata and wraps AI output | Governance Orchestrator Agent | Check Critical Issues Threshold | ## Format Report<br>**What** — Consolidates agent outputs into a structured governance report.<br>**Why** — Ensures consistent, readable output before severity assessment. |
| Check Critical Issues Threshold | n8n-nodes-base.if | Tests critical issue count threshold | Format Final Report | Prepare Escalation Alert; Log Standard Report | ## Severity Routing<br>**What** — Routes findings to Critical or Medium severity handlers.<br>**Why** — Prioritises escalation paths based on risk level automatically.<br>## Format Report<br>**What** — Consolidates agent outputs into a structured governance report.<br>**Why** — Ensures consistent, readable output before severity assessment. |
| Prepare Escalation Alert | n8n-nodes-base.set | Builds escalation payload for high critical-issue count | Check Critical Issues Threshold | Route by Severity Level | ## Severity Routing<br>**What** — Routes findings to Critical or Medium severity handlers.<br>**Why** — Prioritises escalation paths based on risk level automatically. |
| Route by Severity Level | n8n-nodes-base.switch | Routes escalation cases by overall health score | Prepare Escalation Alert | Critical Severity Handler; Medium Severity Handler | ## Severity Routing<br>**What** — Routes findings to Critical or Medium severity handlers.<br>**Why** — Prioritises escalation paths based on risk level automatically. |
| Critical Severity Handler | n8n-nodes-base.set | Tags report as critical severity | Route by Severity Level | Aggregate Critical Findings | ## Severity Routing<br>**What** — Routes findings to Critical or Medium severity handlers.<br>**Why** — Prioritises escalation paths based on risk level automatically. |
| Medium Severity Handler | n8n-nodes-base.set | Tags report as medium severity | Route by Severity Level | Aggregate Critical Findings | ## Severity Routing<br>**What** — Routes findings to Critical or Medium severity handlers.<br>**Why** — Prioritises escalation paths based on risk level automatically. |
| Aggregate Critical Findings | n8n-nodes-base.aggregate | Aggregates escalated severity outputs | Critical Severity Handler; Medium Severity Handler | Merge Analysis Paths | ## Aggregate, Merge & Deliver<br>**What** — Aggregates critical findings, merges all analysis paths, enriches output, and logs the standard report.<br>**Why** — Unifies parallel outputs into a single audit-ready deliverable with full contextual detail. |
| Log Standard Report | n8n-nodes-base.set | Records metadata for non-escalated reports | Check Critical Issues Threshold | Merge Analysis Paths | ## Aggregate, Merge & Deliver<br>**What** — Aggregates critical findings, merges all analysis paths, enriches output, and logs the standard report.<br>**Why** — Unifies parallel outputs into a single audit-ready deliverable with full contextual detail. |
| Merge Analysis Paths | n8n-nodes-base.merge | Rejoins escalated and standard-report branches | Log Standard Report; Aggregate Critical Findings | Enrich Final Output | ## Aggregate, Merge & Deliver<br>**What** — Aggregates critical findings, merges all analysis paths, enriches output, and logs the standard report.<br>**Why** — Unifies parallel outputs into a single audit-ready deliverable with full contextual detail. |
| Enrich Final Output | n8n-nodes-base.set | Appends final workflow metadata | Merge Analysis Paths |  | ## Aggregate, Merge & Deliver<br>**What** — Aggregates critical findings, merges all analysis paths, enriches output, and logs the standard report.<br>**Why** — Unifies parallel outputs into a single audit-ready deliverable with full contextual detail. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Prerequisites<br>- OpenAI or compatible LLM API credentials<br>- Git repository access (GitHub, GitLab, or Bitbucket API)<br>- Notification channel (Slack, email, or webhook)<br>## Use Cases<br>- Automated pre-release security and compliance audits<br>## Customisation<br>- Adjust severity thresholds to match internal risk frameworks<br>## Benefits<br>- Eliminates manual code audit effort across engineering teams |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas setup guidance |  |  | ## Setup Steps<br>1. Configure `Extract Repository Metadata` with your Git provider or repository API credentials.<br>2. Set severity thresholds in the `Check Critical Issues Threshold` node to match your governance policy.<br>3. Configure `Prepare Escalation Alert` with your notification channel. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas high-level workflow explanation |  |  | ## How It Works<br>This workflow automates end-to-end code repository governance scanning using a multi-agent AI orchestration system. Designed for engineering leads, DevSecOps teams, and CTOs, it replaces manual code audits with a structured, AI-driven compliance and security analysis pipeline. The workflow begins by extracting repository metadata, which is passed to a Governance Orchestrator Agent coordinating four specialised sub-agents: Static Code Analysis, Architectural Compliance, CTO Report Generation, and Security Vulnerability Analysis. Outputs are consolidated into a Structured Governance Output, formatted as a final report, then routed by severity level. Critical findings trigger escalation alerts and are aggregated separately, while medium findings are handled independently. All paths converge to merge analysis results, enrich the final output, and deliver a board-ready governance report with full audit traceability. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for severity routing block |  |  | ## Severity Routing<br>**What** — Routes findings to Critical or Medium severity handlers.<br>**Why** — Prioritises escalation paths based on risk level automatically. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for report formatting block |  |  | ## Format Report<br>**What** — Consolidates agent outputs into a structured governance report.<br>**Why** — Ensures consistent, readable output before severity assessment. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas note for extraction/orchestration block |  |  | ## Extract, Orchestrator & Sub-Agents<br>**What** — Coordinates static code, architecture, CTO report, and security agents.<br>**Why** — Decomposes governance into specialised tasks for higher accuracy. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas note for aggregation and delivery block |  |  | ## Aggregate, Merge & Deliver<br>**What** — Aggregates critical findings, merges all analysis paths, enriches output, and logs the standard report.<br>**Why** — Unifies parallel outputs into a single audit-ready deliverable with full contextual detail. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named:  
   `Smart code governance scan with severity routing and compliance report`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Name it: `Start Governance Scan`

3. **Add an SSH node**
   - Node type: **SSH**
   - Name it: `Extract Repository Metadata`
   - Configure SSH credentials to reach the system where the repository is accessible.
   - Set the command to:
     ```bash
     find {{ $json.repositoryPath }} -type f \( -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.java" -o -name "*.go" \) | head -100
     ```
   - Connect `Start Governance Scan` → `Extract Repository Metadata`

4. **Create the orchestrator model**
   - Node type: **OpenAI Chat Model**
   - Name it: `Orchestrator Model`
   - Credential: OpenAI API credential
   - Model: `gpt-4o`
   - Temperature: `0.2`

5. **Create the central AI orchestrator**
   - Node type: **AI Agent**
   - Name it: `Governance Orchestrator Agent`
   - Set input text to:
     ```n8n
     {{ $json.codebaseData }}
     ```
   - Enable structured output parser
   - Set the system message to define a software governance orchestrator coordinating:
     - Static code analysis
     - Architecture compliance
     - CTO report synthesis
     - Security scanning
   - Connect:
     - `Extract Repository Metadata` → `Governance Orchestrator Agent`
     - `Orchestrator Model` → AI language model input of `Governance Orchestrator Agent`

6. **Create the static code analysis tool**
   - Node type: **AI Agent Tool**
   - Name it: `Static Code Analysis Agent`
   - Tool input:
     ```n8n
     {{ $fromAI("codeFiles", "The code files and repository structure to analyze", "string") }}
     ```
   - Enable output parser
   - Add a system message instructing analysis of:
     - SOLID violations
     - anti-patterns
     - code smells
     - maintainability risks
     - technical debt index
     - severity and remediation by file/line
   - Add a clear tool description.
   - Connect its AI tool output to `Governance Orchestrator Agent`

7. **Create the static analysis model**
   - Node type: **OpenAI Chat Model**
   - Name it: `Static Analysis Model`
   - Credential: same OpenAI credential
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Connect to the AI language model input of `Static Code Analysis Agent`

8. **Create the architecture analysis tool**
   - Node type: **AI Agent Tool**
   - Name it: `Architectural Compliance Agent`
   - Tool input:
     ```n8n
     {{ $fromAI("architectureData", "The repository structure and architectural metadata to validate", "string") }}
     ```
   - Enable output parser
   - Set the system message to analyze:
     - microservices boundaries
     - API contracts
     - dependency stability
     - scalability constraints
     - resilience patterns
   - Connect its AI tool output to `Governance Orchestrator Agent`

9. **Create the architecture model**
   - Node type: **OpenAI Chat Model**
   - Name it: `Architectural Analysis Model`
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Connect to `Architectural Compliance Agent`

10. **Create the executive reporting tool**
    - Node type: **AI Agent Tool**
    - Name it: `CTO Report Generation Agent`
    - Tool input:
      ```n8n
      {{ $fromAI("analysisResults", "The combined technical analysis results from code and architecture agents", "string") }}
      ```
    - Enable output parser
    - System message should request:
      - risk matrix
      - remediation backlog prioritization
      - compliance summary
      - business framing, ROI, timelines
    - Connect its AI tool output to `Governance Orchestrator Agent`

11. **Create the executive reporting model**
    - Node type: **OpenAI Chat Model**
    - Name it: `Report Generation Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect to `CTO Report Generation Agent`

12. **Create the security scanning tool**
    - Node type: **AI Agent Tool**
    - Name it: `Security Vulnerability Scanner Agent`
    - Tool input:
      ```n8n
      {{ $fromAI('securityContext', 'The code and dependency information to scan for security vulnerabilities', 'string') }}
      ```
    - System message should cover:
      - injection flaws
      - auth weaknesses
      - hardcoded secrets
      - insecure dependencies
      - OWASP Top 10 mapping
      - CVE/CVSS guidance
    - Connect its AI tool output to `Governance Orchestrator Agent`

13. **Create the security model**
    - Node type: **OpenAI Chat Model**
    - Name it: `Security Analysis Model`
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Connect to `Security Vulnerability Scanner Agent`

14. **Create the structured output parser**
    - Node type: **Structured Output Parser**
    - Name it: `Structured Governance Output`
    - Use a manual schema with these fields:
      - `riskMatrix`: array of objects with `category`, `impact`, `likelihood`, `severity`
      - `remediationBacklog`: array of objects with `issue`, `priority`, `effort`, `businessImpact`, `timeline`
      - `complianceSummary`: object with numeric fields `technicalDebtIndex`, `architectureComplianceScore`, `overallHealthScore`, `criticalIssuesCount`, `highPriorityCount`
      - `executiveSummary`: string
      - `recommendations`: array of strings
    - Connect parser output to `Governance Orchestrator Agent`

15. **Add a Set node for final report formatting**
    - Node type: **Set**
    - Name it: `Format Final Report`
    - Keep existing fields
    - Add:
      - `reportGeneratedAt` = `{{ $now.toISO() }}`
      - `repositoryScanned` = `{{ $('Extract Repository Metadata').item.json.stdout }}`
      - `governanceReport` = `{{ JSON.stringify($json) }}`
    - Connect `Governance Orchestrator Agent` → `Format Final Report`

16. **Add an IF node for critical threshold**
    - Node type: **If**
    - Name it: `Check Critical Issues Threshold`
    - Condition:
      - Left value:
        ```n8n
        {{ $('Format Final Report').item.json.governanceReport.complianceSummary.criticalIssuesCount }}
        ```
      - Operator: greater than
      - Right value: `5`
    - Connect `Format Final Report` → `Check Critical Issues Threshold`

17. **Add a Set node for escalations**
    - Node type: **Set**
    - Name it: `Prepare Escalation Alert`
    - Add:
      - `alertType` = `CRITICAL_THRESHOLD_EXCEEDED`
      - `criticalIssuesCount` = `{{ $json.complianceSummary.criticalIssuesCount }}`
      - `escalationRequired` = `true`
      - `alertTimestamp` = `{{ $now.toISO() }}`
      - `fullReport` = `{{ JSON.stringify($json) }}`
    - Connect the **true** output of `Check Critical Issues Threshold` to this node

18. **Add a Switch node for severity routing**
    - Node type: **Switch**
    - Name it: `Route by Severity Level`
    - Rule 1:
      - `{{ $json.complianceSummary.overallHealthScore }} < 30`
    - Rule 2:
      - `{{ $json.complianceSummary.overallHealthScore }} < 70`
    - Connect `Prepare Escalation Alert` → `Route by Severity Level`

19. **Add the critical severity handler**
    - Node type: **Set**
    - Name it: `Critical Severity Handler`
    - Add:
      - `severityLevel` = `CRITICAL`
      - `actionRequired` = `IMMEDIATE_REMEDIATION`
      - `notificationPriority` = `P1`
      - `escalationPath` = `CTO_DIRECT`
    - Connect output 0 of `Route by Severity Level` to this node

20. **Add the medium severity handler**
    - Node type: **Set**
    - Name it: `Medium Severity Handler`
    - Add:
      - `severityLevel` = `MEDIUM`
      - `actionRequired` = `SCHEDULED_REMEDIATION`
      - `notificationPriority` = `P2`
      - `escalationPath` = `ENGINEERING_LEAD`
    - Connect output 1 of `Route by Severity Level` to this node

21. **Add an Aggregate node**
    - Node type: **Aggregate**
    - Name it: `Aggregate Critical Findings`
    - Aggregate mode: **Aggregate All Item Data**
    - Connect:
      - `Critical Severity Handler` → `Aggregate Critical Findings`
      - `Medium Severity Handler` → `Aggregate Critical Findings`

22. **Add the standard logging node**
    - Node type: **Set**
    - Name it: `Log Standard Report`
    - Add:
      - `reportStatus` = `STANDARD_COMPLIANCE`
      - `loggedAt` = `{{ $now.toISO() }}`
      - `summary` = `{{ $json.executiveSummary }}`
    - Connect the **false** output of `Check Critical Issues Threshold` to this node

23. **Add a Merge node**
    - Node type: **Merge**
    - Name it: `Merge Analysis Paths`
    - Connect:
      - `Log Standard Report` → input 0
      - `Aggregate Critical Findings` → input 1

24. **Add the final enrichment node**
    - Node type: **Set**
    - Name it: `Enrich Final Output`
    - Add:
      - `workflowCompletedAt` = `{{ $now.toISO() }}`
      - `totalAgentsInvolved` = `5`
      - `analysisDepth` = `COMPREHENSIVE_MULTI_AGENT`
      - `frameworkVersion` = `2.0-ENHANCED`
    - Connect `Merge Analysis Paths` → `Enrich Final Output`

25. **Add optional sticky notes**
    - Add notes for prerequisites, setup, severity routing, formatting, orchestration, and aggregation if you want the same canvas documentation.

26. **Configure credentials**
    - **OpenAI credential:** required for all four model nodes plus orchestrator model.
    - **SSH credential:** required for repository metadata extraction.
    - The sticky notes mention Git provider and notification channel credentials, but this JSON does not include actual GitHub/GitLab/Bitbucket or Slack/email/webhook nodes.

27. **Important implementation corrections recommended before production**
    - Replace orchestrator input `{{ $json.codebaseData }}` with actual SSH output, likely:
      ```n8n
      {{ $json.stdout }}
      ```
      or map it explicitly in an intermediate Set node.
    - Do **not** stringify `governanceReport` if downstream nodes need object access. Store it directly as JSON/object instead.
    - Fix `Check Critical Issues Threshold` to read from actual structured fields, for example:
      ```n8n
      {{ $json.complianceSummary.criticalIssuesCount }}
      ```
    - Add a default/fallback route in `Route by Severity Level` for `overallHealthScore >= 70`.
    - If real code analysis is required, add file content extraction, not just path listing.

28. **Test with sample input**
    - Start the workflow manually.
    - Provide input JSON containing at minimum:
      ```json
      {
        "repositoryPath": "/path/to/repository"
      }
      ```
    - Confirm the SSH node returns file paths and that downstream AI nodes receive valid text.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI or compatible LLM API credentials are required. | Prerequisites |
| Git repository access is listed as a prerequisite, but the actual implementation uses SSH and not a Git provider node. | Prerequisites / implementation gap |
| Notification channel is listed as a prerequisite, but no Slack, email, or webhook delivery node exists in the workflow. | Prerequisites / implementation gap |
| Automated pre-release security and compliance audits are highlighted as a core use case. | Use cases |
| Severity thresholds should be adjusted to match internal risk frameworks. | Customisation |
| The workflow is positioned as a replacement for manual code audits across engineering teams. | Benefits |
| The workflow claims “board-ready governance report with full audit traceability,” but no external persistence or notification step is present. | Delivery expectation vs current implementation |
| The analysis currently relies on repository file paths discovered by SSH; it does not fetch actual file contents or dependency manifests. | Important architectural limitation |