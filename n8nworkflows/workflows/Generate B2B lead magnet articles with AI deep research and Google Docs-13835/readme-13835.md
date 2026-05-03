Generate B2B lead magnet articles with AI deep research and Google Docs

https://n8nworkflows.xyz/workflows/generate-b2b-lead-magnet-articles-with-ai-deep-research-and-google-docs-13835


# Generate B2B lead magnet articles with AI deep research and Google Docs

This document provides a technical breakdown and reconstruction guide for the **Generate B2B Lead Magnet Articles** workflow. This system automates the transformation of a simple topic into a 4,000-word, research-backed, formatted Google Document.

---

### 1. Workflow Overview

The workflow follows a "Research-First" architectural pattern. Instead of asking an AI to write an entire article at once (which leads to hallucinations and shallow content), it breaks the topic into five strategic pillars, researches them independently in parallel, and then synthesizes the results into a high-authority B2B lead magnet.

**Logical Blocks:**
*   **1.1 Strategic Ideation:** Receives a topic and decomposes it into five distinct research queries.
*   **1.2 Distributed Research:** Executes five parallel AI agents to find data, stats, and frameworks for each specific query.
*   **1.3 Content Synthesis:** Aggregates all research data and uses a "Head Editor" agent to write the full-length article.
*   **1.4 Document Engineering:** Creates a Google Doc and programmatically applies formatting (headings, bold text) based on custom tags.
*   **1.5 Delivery & Analytics:** Sets permissions, provides access links via chat, and logs the execution to a spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Strategic Ideation
This block ensures the article has a professional B2B "angle" rather than being generic.
*   **Nodes:** `Submit Your Topic`, `Refine into 5 Strategic Queries`, `Validate Queries`.
*   **Configuration:** The AI Agent is restricted to outputting a specific JSON schema containing the target persona, article angle, and five optimized search queries.
*   **Key Logic:** The `Validate Queries` node acts as a safety net. If the AI fails to produce valid JSON, it falls back to a hardcoded structure to prevent workflow failure. It also injects the `authorName`.

#### 2.2 Distributed Research
This block handles the heavy lifting of data gathering.
*   **Nodes:** `Split into 5 Research Tasks`, `Deep Web Researcher`, `Parse Research Output`.
*   **Technical Role:** The `Split Out` node takes the list of 5 queries and creates 5 separate execution branches. 
*   **Agent Logic:** The `Deep Web Researcher` uses a language model (configured for Ollama by default) to generate 400–600 words per section, including specific statistics and company examples.

#### 2.3 Content Synthesis
*   **Nodes:** `Build TOC and Compile Research`, `Final Editor and Polish`.
*   **Configuration:** The `Build TOC` node uses JavaScript to sort the parallel results back into their original order (q1 through q5). 
*   **Writing Strategy:** The `Final Editor` uses a "Plain Text with Tags" approach. Instead of Markdown, it uses `{BOLD}text{/BOLD}` and `{H3}text{/H3}` tags, which are easier to process for the Google Docs API later.

#### 2.4 Document Engineering
This is the most technically complex part of the workflow, moving from text to a formatted file.
*   **Nodes:** `Prepare Document Content`, `Create Google Doc`, `Extract Doc ID`, `Build Format Requests`, `Apply Formatting to Doc`.
*   **Google Docs API Logic:** 
    *   First, an HTTP Request creates a blank document.
    *   The `Build Format Requests` node calculates the exact `startIndex` and `endIndex` for every word wrapped in `{BOLD}` or `{H3}` tags.
    *   It then sends a `batchUpdate` request to the API to apply styles. This circumvents the limitation where standard API text insertion loses all formatting.

#### 2.5 Delivery & Analytics
*   **Nodes:** `Make Doc Shareable`, `Generate Chat Response`, `Log to Tracking Sheet`.
*   **Outcome:** The Google Drive API is called to set the file permission to "anyone with the link can read." The final chat message provides the user with Edit, Preview, and PDF download links.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Submit Your Topic** | Chat Trigger | Entry Point | - | Refine Queries | User submits a topic via chat. |
| **Refine into 5 Strategic Queries** | AI Agent | Strategy Generation | Submit Your Topic | Validate Queries | AI refines it into 5 strategic research queries with SEO terms and angles. |
| **Validate Queries** | Code | Data Sanitization | Refine Queries | Split Tasks | Update YOUR_AUTHOR_NAME in this node before running. |
| **Split into 5 Research Tasks** | Split Out | Parallelization | Validate Queries | Deep Researcher | Each query is researched independently. |
| **Deep Web Researcher** | AI Agent | Data Gathering | Split Tasks | Parse Research | Produces stats, examples, and frameworks (400–600 words each). |
| **Parse Research Output** | Code | JSON Cleanup | Deep Researcher | Build TOC | Part of Parallel Deep Research block. |
| **Build TOC and Compile Research** | Code | Synthesis | Parse Research | Final Editor | All research is merged into a structured document. |
| **Final Editor and Polish** | AI Agent | Long-form Writing | Build TOC | Prepare Doc Content | Produces the complete 2,500–4,000 word lead magnet with formatting tags. |
| **Prepare Document Content** | Code | Text Cleaning | Final Editor | Create Google Doc | Removes AI thinking text and cleans up markdown. |
| **Create Google Doc** | HTTP Request | File Creation | Prepare Doc Content | Extract Doc ID | Article is created as a Google Doc via API. |
| **Extract Doc ID** | Code | Variable Extraction | Create Google Doc | Build Format Requests | Extracts ID for subsequent API calls. |
| **Build Format Requests** | Code | Index Calculation | Extract Doc ID | Apply Formatting | Bold text and headings are applied programmatically. |
| **Apply Formatting to Doc** | HTTP Request | Style Application | Build Format Requests | Make Shareable | Executes the batchUpdate for styling. |
| **Make Doc Shareable** | HTTP Request | Permission Mgmt | Apply Formatting | Chat Response, Log | Public sharing is enabled automatically. |
| **Generate Chat Response** | Code | UX/Delivery | Make Shareable | - | Returns edit, view, and PDF download links to the user. |
| **Log to Tracking Sheet** | Google Sheets | Audit Trail | Make Shareable | - | Logs article metadata to Google Sheets for tracking. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Configure credentials for: **Google Docs API** (OAuth2), **Google Drive API** (OAuth2), and **Google Sheets API**.
    *   Set up an AI provider (Ollama is default; OpenAI or Anthropic can be used by swapping the Language Model node).

2.  **The Trigger & Strategy:**
    *   Create a **Chat Trigger** node.
    *   Add an **AI Agent** node. Use a System Message that demands a JSON output containing an array of 5 queries (q1-q5).
    *   Add a **Code Node** to parse the AI output and add a hardcoded "Author Name" variable.

3.  **The Research Loop:**
    *   Use a **Split Out** node to target the array of queries.
    *   Add a second **AI Agent** ("Researcher"). Set its prompt to generate exactly one chapter of the article based on the query it receives.
    *   Add a **Code Node** to standardize the output from the researcher (ensuring fields like `sectionContent` and `sources` exist).

4.  **The Compilation:**
    *   Add a **Code Node** that uses `$input.all()` to gather all results from the previous step. Sort them by Query ID.
    *   Add a third **AI Agent** ("Final Editor"). Provide it with all the gathered research. Instruct it to use `{BOLD}` for emphasis and `{H3}` for headers.

5.  **Google Docs Integration:**
    *   **HTTP Request (POST):** Endpoint `https://docs.googleapis.com/v1/documents`. Body: `{"title": "..."}`.
    *   **Code Node:** Use regex to find the start and end positions of all `{BOLD}` and `{H3}` tags in the text. Store these as an array of "requests."
    *   **HTTP Request (POST):** Endpoint `https://docs.googleapis.com/v1/documents/{{docId}}:batchUpdate`. Pass the array of formatting requests.

6.  **Sharing & Finalizing:**
    *   **HTTP Request (POST):** Endpoint `https://www.googleapis.com/drive/v3/files/{{docId}}/permissions`. Body: `{"role": "reader", "type": "anyone"}`.
    *   **Google Sheets Node:** Append a row with the Title, URL, and Date.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Author Customization** | Users must update the `Validate Queries` node with their own name to avoid "YOUR_AUTHOR_NAME" appearing in docs. |
| **Ollama Model Name** | The default model is a placeholder. Update the Ollama node with a model like `llama3` or `mistral`. |
| **Formatting Logic** | The workflow uses index-based formatting because Google Docs API does not natively support Markdown conversion via simple text insertion. |
| **Word Count Management** | The 5-agent split is designed to bypass LLM context limits and "laziness" often seen in single-prompt long-form requests. |