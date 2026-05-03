Generate AI research papers with Claude, arXiv, Google Scholar and DOCX export

https://n8nworkflows.xyz/workflows/generate-ai-research-papers-with-claude--arxiv--google-scholar-and-docx-export-13713


# Generate AI research papers with Claude, arXiv, Google Scholar and DOCX export

### 1. Workflow Overview

This workflow is an advanced academic document generator that automates the creation of professional research papers. It utilizes a multi-agent architecture to handle the complex requirements of scholarly writing, including literature review, structured section drafting, and bibliography management.

The process is divided into four main logical phases:
- **1.1 Research & Data Retrieval:** Capturing user input (title, abstract, keywords) and querying the arXiv API and Google Scholar to ground the paper in existing academic literature.
- **1.2 Knowledge Synthesis:** Consolidating references into a unified context for the AI writing agents.
- **1.3 Multi-Agent Orchestration:** A central "Orchestrator" agent delegates the writing of specific sections (Introduction, Related Work, Methodology, Results, Discussion, and Conclusion) to specialized sub-agents.
- **1.4 Assembly & Export:** Merging the generated sections with an automatically formatted APA-style bibliography and converting the result into a downloadable DOCX file.

---

### 2. Block-by-Block Analysis

#### 2.1 Research Input & Web Search
This block handles the initial data gathering. It takes a user's research concept and finds relevant supporting documents.

*   **Nodes Involved:** `Research Paper Input Form`, `Workflow Configuration`, `Search arXiv API`, `Parse and Format References`.
*   **Node Details:**
    *   **Research Paper Input Form:** Trigger node (Form) collecting `paperTitle`, `abstract`, and `researchKeywords`.
    *   **Workflow Configuration:** Sets technical constants like `targetReferences` (35) and the arXiv API URL.
    *   **Search arXiv API:** Executes an HTTP request to arXiv using the paper title.
    *   **Parse and Format References:** A Code node that parses the arXiv XML response into a clean JSON structure containing titles, authors, years, and summaries.
    *   **Edge Cases:** No results from arXiv or malformed XML; handled by JS logic within the Code node to return empty arrays rather than crashing.

#### 2.2 Research Gathering Agent
This block uses AI to expand the literature search beyond arXiv to include broader databases like IEEE Xplore or ACM Digital Library.

*   **Nodes Involved:** `Research Gathering Agent`, `Claude Sonnet Model`, `Google Scholar Search Tool`, `Research Output Parser`.
*   **Node Details:**
    *   **Research Gathering Agent:** An AI Agent node using Claude 3.5 Sonnet to find 35 total high-quality references.
    *   **Google Scholar Search Tool:** SerpApi integration allowing the agent to browse live scholarly results.
    *   **Research Output Parser:** Forces the agent to return a structured list of references (title, author, year, relevance).

#### 2.3 Paper Orchestration & Section Writing
The core of the workflow. A central orchestrator manages six parallel-capable sub-agents, each specialized in one specific academic section.

*   **Nodes Involved:** `Paper Orchestrator Agent`, `Claude Model for Orchestrator`, `Orchestrator Output Parser`, and 6 tool-based writer agents (Introduction through Conclusion).
*   **Node Details:**
    *   **Section Writer Agent Tools:** Each tool (e.g., `Methodology Writer Agent Tool`) has a specific `systemMessage` defining word count (e.g., 1200-1500 words for Methodology) and specific academic requirements.
    *   **Claude Models (Multiple):** Each writing tool is backed by its own Claude 3.5 Sonnet instance to maintain high context windows and specialized temperatures (0.4 for writing).
    *   **Output Parsers:** Each section has a Structured Output Parser to ensure the agent returns both the "content" (string) and the "citations" (array).
    *   **Orchestrator:** Coordinates the tools, passing the same context (Title, Abstract, References) to ensure consistency across the manuscript.

#### 2.4 Assemble, Bibliography & DOCX Export
The final phase reconstructs the separate AI outputs into a single cohesive document.

*   **Nodes Involved:** `Assemble Paper Sections`, `Generate Bibliography`, `Combine Paper and Bibliography`, `Format as DOCX Content`, `Generate DOCX File`.
*   **Node Details:**
    *   **Assemble Paper Sections:** Code node that concatenates strings into a structured text body with numbered headings.
    *   **Generate Bibliography:** Code node that takes all references, removes duplicates based on title, sorts them alphabetically by author, and formats them into APA style.
    *   **Format as DOCX Content:** Prepares the final string with a title page, abstract, main sections, and the bibliography.
    *   **Generate DOCX File:** Converts the text content into a binary `.docx` file using n8n's "Convert to File" node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Research Paper Input Form | n8n-nodes-base.formTrigger | Input Reception | None | Workflow Configuration | Research Input & Web Search with Formatting |
| Workflow Configuration | n8n-nodes-base.set | Parameter Setup | Research Paper Input Form | Search arXiv API | Research Input & Web Search with Formatting |
| Search arXiv API | n8n-nodes-base.httpRequest | Data Retrieval | Workflow Configuration | Parse and Format References | Research Input & Web Search with Formatting |
| Parse and Format References | n8n-nodes-base.code | Data Processing | Search arXiv API | Research Gathering Agent | Research Input & Web Search with Formatting |
| Research Gathering Agent | @n8n/n8n-nodes-langchain.agent | AI Research | Parse and Format References | Paper Orchestrator, Gen. Bibliography | Research Gathering Agent |
| Paper Orchestrator Agent | @n8n/n8n-nodes-langchain.agent | Task Delegation | Research Gathering Agent | Assemble Paper Sections | Paper Orchestration Agent |
| Intro/Related Work/etc. Tools | @n8n/n8n-nodes-langchain.agentTool | Section Drafting | Orchestrator Agent | Orchestrator Agent | Section Writing Agents |
| Assemble Paper Sections | n8n-nodes-base.code | Text Compilation | Paper Orchestrator Agent | Combine Paper/Biblio | Assemble, Bibliography & DOCX Export |
| Generate Bibliography | n8n-nodes-base.code | Citation Formatting | Research Gathering Agent | Combine Paper/Biblio | Assemble, Bibliography & DOCX Export |
| Combine Paper and Bibliography | n8n-nodes-base.merge | Data Merging | Assemble Sections, Gen. Bibliography | Format as DOCX Content | Assemble, Bibliography & DOCX Export |
| Format as DOCX Content | n8n-nodes-base.code | Structural Formatting | Combine Paper/Biblio | Generate DOCX File | Assemble, Bibliography & DOCX Export |
| Generate DOCX File | n8n-nodes-base.convertToFile | Binary Generation | Format as DOCX Content | None | Assemble, Bibliography & DOCX Export |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger & Search Setup:**
    *   Create a **Form Trigger** with fields: `paperTitle`, `abstract` (textarea), `researchKeywords`.
    *   Connect a **Set node** to define constants: `targetReferences` (35) and `arxivSearchUrl`.
    *   Add an **HTTP Request node** using GET to the arXiv API. Use query parameters for `search_query` (formatting title with '+') and `max_results`.
    *   Add a **Code node** to parse the XML from arXiv into a JSON array of objects.

2.  **Research Agent Setup:**
    *   Create a **LangChain Agent** (Research Gathering Agent).
    *   Attach an **Anthropic Chat Model** (Claude 3.5 Sonnet) and a **SerpApi Tool** (Google Scholar).
    *   Add a **Structured Output Parser** defining the `additionalReferences` array schema.

3.  **Section Writing Architecture:**
    *   Create six **Agent Tool nodes** (Introduction, Related Work, Methodology, Results, Discussion, Conclusion).
    *   For each tool, configure the **System Message** with specific word counts and academic goals. 
    *   Attach a **Claude Model** and a **Structured Output Parser** (returning `content` and `citations`) to each tool.

4.  **Orchestrator Setup:**
    *   Create a **LangChain Agent** (Paper Orchestrator).
    *   Connect the six Writer Tools to this agent.
    *   Configure the system prompt to instruct the agent to call all six tools in sequence or parallel using the input context.
    *   Add an **Orchestrator Output Parser** that expects a JSON object containing all six section strings.

5.  **Final Assembly & Export:**
    *   Create a **Code node** (Assemble Paper Sections) to concatenate the orchestrator's output with markdown-style headers.
    *   Create a second **Code node** (Generate Bibliography) that filters the reference list and applies APA formatting logic (String manipulation for Author, Year, Title).
    *   **Merge** these two streams (Wait for both).
    *   Use a **Code node** to build the final document string (Title Page -> Abstract -> Sections -> References).
    *   Use the **Convert to File node** (Operation: To Binary) to create the `.docx` file using the string content.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Requires SerpApi for Google Scholar and Anthropic API for Claude 3.5 Sonnet. |
| **Customization** | Can swap Claude for GPT-4o or NVIDIA NIM by changing the Model nodes. |
| **Use Case** | Ideal for researchers generating high-quality first drafts for academic submissions. |
| **Author Credit** | Designed for automated scholarly article planning and multi-agent coordination. |