Generate research proposals with GPT-4o, web search, and quality control agents

https://n8nworkflows.xyz/workflows/generate-research-proposals-with-gpt-4o--web-search--and-quality-control-agents-13869


# Generate research proposals with GPT-4o, web search, and quality control agents

This document provides a comprehensive technical reference for the **Smart research proposal generator with multi-agent quality control** workflow.

---

### 1. Workflow Overview

This workflow automates the creation of high-quality academic and professional research proposals through a multi-agent AI architecture. It leverages specialized agents to handle distinct sections of a proposal—ranging from scientific methodology to strategic planning and ethics—ensuring depth and rigor that a single-prompt approach often lacks.

The process is divided into four functional stages:
1.  **Orchestration & Research:** A Supervisor Agent identifies funding trends and coordinates sub-agents.
2.  **Specialized Content Generation:** Sub-agents (Research and Strategy) generate detailed technical and operational content.
3.  **Parsing & Quality Control:** The draft is structured via code and evaluated by a QC Agent against a quality threshold.
4.  **Formatting & Storage:** Depending on the quality score, the proposal is either formatted for final use or flagged for human revision before being logged in a database.

---

### 2. Block-by-Block Analysis

#### 2.1 Orchestration & Research
The "brain" of the workflow, responsible for delegating tasks and gathering external context.

*   **Nodes Involved:** `Start Proposal Generation`, `Supervisor Agent`, `Supervisor Model`, `Conversation Memory`, `Funding Agency Research Tool`, `Web Search Tool`.
*   **Node Details:**
    *   **Supervisor Agent (AI Agent):** Uses a system message to act as a research coordinator. It has access to tools and sub-agents to synthesize a cohesive proposal.
    *   **Supervisor Model (OpenAI Chat Model):** Configured with GPT-4o and a temperature of 0.7 for creative yet structured orchestration.
    *   **Funding Agency Research Tool (HTTP Request Tool):** Fetches real-time AI funding priorities from a specified endpoint (placeholder: `https://api.example.com/ai-funding-trends`).
    *   **Web Search Tool (SerpAPI):** Allows the supervisor to browse the live web for the latest research gaps.
    *   **Conversation Memory (Window Buffer):** Maintains a context window of 10 interactions to ensure consistency throughout the generation process.

#### 2.2 Specialized Content Generation
Specialized sub-agents act as "Tools" for the Supervisor, providing deep-dive technical content.

*   **Nodes Involved:** `Research Content Agent`, `Research Content Model`, `Strategic Planning Agent`, `Strategic Planning Model`, `Ethics Model`, `Impact Model`.
*   **Node Details:**
    *   **Research Content Agent:** Focuses on the "Scientific Foundation." It generates problem statements, hypotheses, and detailed AI methodology (data strategy, model design).
    *   **Strategic Planning Agent:** Focuses on "Operational Components." It handles work plans, budget justifications, and sustainability.
    *   **Ethics & Impact Models:** Specialized GPT-4o instances connected to the Strategic Planning agent to ensure high-fidelity narratives regarding data privacy, bias mitigation, and societal benefits.

#### 2.3 Parsing & Quality Control
Before the proposal is finalized, it must be verified against professional standards.

*   **Nodes Involved:** `Parse Proposal Structure`, `Quality Control Agent`, `Quality Control Model`, `Quality Assessment Output`.
*   **Node Details:**
    *   **Parse Proposal Structure (Code):** A JavaScript snippet that maps the raw string output from the Supervisor into a structured JSON object (Title, Objectives, Budget, etc.).
    *   **Quality Control Agent (AI Agent):** Reviews the JSON proposal. It looks for completeness, scientific rigor, and feasibility.
    *   **Quality Assessment Output (Structured Output Parser):** Enforces a strict JSON schema on the QC Agent, requiring a `quality_score` (0-100) and a list of strengths/weaknesses.

#### 2.4 Conditional Routing & Storage
Determines the fate of the proposal based on its performance score.

*   **Nodes Involved:** `Check Quality Score`, `Format Final Proposal`, `Flag for Revision`, `Prepare Storage Data`, `Store Proposal Results`.
*   **Node Details:**
    *   **Check Quality Score (If):** Checks if `quality_score >= 75`.
    *   **Format Final Proposal (Set):** If passed, sets status to "approved" and extracts the full proposal.
    *   **Flag for Revision (Set):** If failed, sets status to "needs_revision" and attaches the QC agent's specific recommendations.
    *   **Store Proposal Results (Data Table):** Saves the final result (approved or flagged) with a timestamp and unique ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Start Proposal Generation** | Manual Trigger | Trigger | None | Supervisor Agent | Multi-Agent Content Generation |
| **Supervisor Agent** | AI Agent | Orchestrator | Start Proposal Generation | Parse Proposal Structure | Multi-Agent Content Generation |
| **Research Content Agent** | Agent Tool | Content Creation | Supervisor Agent | Supervisor Agent | Multi-Agent Content Generation |
| **Strategic Planning Agent** | Agent Tool | Strategy Creation | Supervisor Agent | Supervisor Agent | Multi-Agent Content Generation |
| **Funding Agency Tool** | HTTP Tool | Data Gathering | Supervisor Agent | Supervisor Agent | Multi-Agent Content Generation |
| **Web Search Tool** | SerpAPI | Data Gathering | Supervisor Agent | Supervisor Agent | Multi-Agent Content Generation |
| **Parse Proposal Structure** | Code | Data Formatting | Supervisor Agent | Quality Control Agent | QC Agent scores output. |
| **Quality Control Agent** | AI Agent | Evaluation | Parse Proposal Structure | Check Quality Score | QC Agent scores output. |
| **Check Quality Score** | If | Routing | Quality Control Agent | Format Final / Flag for Revision | High-scoring proposals are formatted. |
| **Format Final Proposal** | Set | Transformation | Check Quality Score (True) | Prepare Storage Data | High-scoring proposals are formatted. |
| **Flag for Revision** | Set | Transformation | Check Quality Score (False) | Prepare Storage Data | Low-scoring ones are flagged. |
| **Prepare Storage Data** | Set | Metadata Generation | Format Final / Flag for Revision | Store Proposal Results | Maintains output quality. |
| **Store Proposal Results** | Data Table | Storage | Prepare Storage Data | None | Maintains output quality. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Setup Logic:** Create a **Manual Trigger** node.
2.  **Supervisor Config:**
    *   Add an **AI Agent** node (Supervisor). Set prompt type to "Define" and use GPT-4o.
    *   Attach a **Window Buffer Memory** node to the Supervisor.
3.  **Tool Integration:**
    *   Create two **Agent Tool** nodes: `Research Content Agent` and `Strategic Planning Agent`.
    *   Inside each tool, attach a **Chat OpenAI** node (GPT-4o) and define their specific system messages (Scientific vs. Operational).
    *   Connect an **HTTP Request Tool** for funding API and a **SerpAPI** tool for web search to the Supervisor.
4.  **Logic Processing:**
    *   Add a **Code Node** after the Supervisor. Use JavaScript to map the agent's output strings to a JSON object containing keys like `title`, `methodology`, and `budget`.
5.  **Quality Gate:**
    *   Add an **AI Agent** (QC Agent). Attach a **Structured Output Parser** node to it. 
    *   Define the schema in the parser to include `quality_score` (number) and `recommendations` (array).
6.  **Routing & Finalization:**
    *   Add an **If Node**. Set the condition to check the `quality_score` from the QC agent.
    *   Create two **Set Nodes** branching from the If node (Approved vs. Revision).
    *   Add a final **Set Node** to generate a `timestamp` and a `proposal_id` (using `$runIndex`).
    *   Connect to a **Data Table** or **Google Sheets** node to save the results.
7.  **Credentials:** Ensure OpenAI API keys are configured in all Model nodes and SerpAPI keys are added to the Search tool.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Time Savings** | Cuts proposal drafting time by 70–80% through parallel specialization. |
| **Model Customization** | Swap AI models (e.g., use GPT-4o-mini for sub-agents) to balance cost and performance. |
| **Prerequisites** | Requires a Web search API key (SerpAPI/Tavily) and a storage destination (Google Sheets/Database). |
| **Human-in-the-loop** | Proposals below a score of 75 are flagged for revision, ensuring human oversight for complex drafts. |