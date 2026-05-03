Collaborative Sales Planning with Multi-Agent AI, Google Docs, and Slack

https://n8nworkflows.xyz/workflows/collaborative-sales-planning-with-multi-agent-ai--google-docs--and-slack-7504


# Collaborative Sales Planning with Multi-Agent AI, Google Docs, and Slack

### 1. Workflow Overview

This workflow, titled **Collaborative Sales Planning with Multi-Agent AI, Google Docs, and Slack**, is designed to automate the creation of a concise, multi-departmental sales season plan. It targets business teams or consultants who want to generate a structured sales plan by orchestrating multiple AI agents representing different company departments (Marketing, Operations, Finance). The output is a polished, shareable PDF document uploaded automatically to Slack for stakeholder review.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures company-specific sales planning parameters manually.
- **1.2 CEO Agent (Orchestrator):** Reads the input brief and delegates tasks to three specialized AI department agents.
- **1.3 Department Agents:** Marketing, Operations, and Finance agents generate focused JSON outputs with their respective plans.
- **1.4 Merge & Document Composition:** The CEO Agent merges and reconciles departmental outputs into a unified Markdown plan.
- **1.5 Document Creation & Export:** Converts Markdown into a Google Doc, then exports it as a PDF.
- **1.6 Sharing:** Uploads the resulting PDF to a Slack channel for team visibility and feedback.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow via manual trigger and collects all necessary input fields that define the sales planning scope.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Edit Fields (Set node for input parameters)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual user action for controlled testing and iteration.  
  - Configuration: No parameters; simply triggers the flow.  
  - Input: None  
  - Output: Triggers the Edit Fields node.  
  - Edge cases: None significant; relies on manual user start.

- **Edit Fields**  
  - Type: Set node  
  - Role: Defines and assigns static input values representing the sales planning brief, e.g., company name, product, sales window, channels, audience, and success metrics.  
  - Configuration: Hardcoded values such as:  
    - Company: "AutomateWithMe"  
    - Product: "Automation Services"  
    - Sale window: "Aug to December annually"  
    - Primary channels: "Amazon, Ebay"  
    - Success metrics: "Revenue"  
    - Audience: "SME's owner (CEO, CFO, CIO)"  
  - Input: Triggered by Manual Trigger  
  - Output: Passes JSON object with these fields to the CEO Agent.  
  - Edge cases: Static values limit flexibility; ideally replaced by dynamic input forms or API triggers for production.  
  - Notes: Input validation is minimal; recommended to keep inputs concise.

---

#### 2.2 CEO Agent (Orchestrator)

**Overview:**  
This AI agent acts as the workflow’s brain, reading the input brief and delegating tailored tasks to three specialist agents (Marketing, Operations, Finance). It then merges their JSON outputs into a consolidated sales plan.

**Nodes Involved:**  
- CEO Agent (LangChain Agent)  
- OpenAI Chat Model (for CEO Agent’s LLM backend)

**Node Details:**

- **CEO Agent**  
  - Type: LangChain AI Tool Agent (multi-tool orchestration)  
  - Role: Orchestrator that:  
    - Reads the user brief,  
    - Calls each department agent exactly once with only necessary info,  
    - Merges outputs, resolves conflicts prioritizing feasibility,  
    - Produces final Markdown and JSON plan outputs.  
  - Configuration:  
    - System prompt defines CEO role, available tools (MarketingDept, OpsDept, FinanceDept), workflow steps, and output format (Markdown sections with Summary, Timeline, Marketing Plan, etc.).  
    - Text prompt dynamically injects input brief fields from previous node.  
  - Input: JSON brief from Edit Fields  
  - Output: JSON containing both Markdown plan (human-readable) and structured JSON plan.  
  - Connections: Calls Marketing Agent, Operations Agent, Finance Agent via AI Tool connections.  
  - Edge cases:  
    - Failure if any department agent fails to respond or returns invalid JSON.  
    - Prompt length or token limits may truncate outputs.  
    - Conflict resolution logic depends on prompt design, may require iteration.  
  - Version-specific: Requires n8n version supporting LangChain AI Tool nodes (v2+).

- **OpenAI Chat Model**  
  - Type: LangChain AI Language Model Node (OpenAI GPT-4.1-mini)  
  - Role: Provides LLM backend for CEO Agent.  
  - Credentials: OpenAI API key configured.  
  - Input: Receives prompts from CEO Agent node.  
  - Output: Returns AI-generated completions to CEO Agent.  
  - Edge cases: API rate limits, authentication failures, network timeouts.

---

#### 2.3 Department Agents (Marketing, Operations, Finance)

**Overview:**  
Each agent specializes in a department’s domain, receiving a slimmed version of the brief from the CEO Agent and returning focused JSON outputs.

**Nodes Involved:**  
- Marketing Agent (LangChain Agent Tool)  
- Operations Agent (LangChain Agent Tool)  
- Finance Agent (LangChain Agent Tool)  
- OpenAI Chat Model1, OpenAI Chat Model2, OpenAI Chat Model3 (LLM backends for each agent)

**Node Details:**

- **Marketing Agent**  
  - Type: LangChain AI Tool Agent  
  - Role: Generates up to 3 practical marketing campaigns, content calendar, and notes.  
  - Configuration:  
    - Text prompt references input from CEO Agent.  
    - Tool description specifies JSON output schema with campaigns, content calendar, and notes.  
  - Input: Slimmed brief from CEO Agent.  
  - Output: JSON with campaign details, calendar entries.  
  - Connection: Uses OpenAI Chat Model1 as LLM backend.  
  - Edge cases: Incorrect JSON formatting, unrealistic campaign proposals, missing KPIs.

- **Operations Agent**  
  - Type: LangChain AI Tool Agent  
  - Role: Assesses inventory/staffing readiness, fulfillment steps, and risks.  
  - Configuration:  
    - JSON output schema includes inventory notes, staffing plans, fulfillment steps, operational risks.  
  - Input: Products, sale window, and constraints from CEO Agent.  
  - Output: Structured JSON with operational plans.  
  - Connection: Uses OpenAI Chat Model2.  
  - Edge cases: Overly optimistic readiness assessment, missing owners or due dates in steps.

- **Finance Agent**  
  - Type: LangChain AI Tool Agent  
  - Role: Recommends discounts, budget splits, revenue targets.  
  - Configuration:  
    - JSON schema specifies discounts array, budget splits, targets, and notes.  
  - Input: Campaign ideas and operational constraints from CEO Agent.  
  - Output: JSON with pricing and budget info.  
  - Connection: Uses OpenAI Chat Model3.  
  - Edge cases: Budget totals exceeding realistic limits, inconsistent discount types.

- **OpenAI Chat Model1/2/3**  
  - Type: LangChain AI Language Model nodes  
  - Role: Provide GPT-4.1-mini completions for respective agents.  
  - Credentials: Shared OpenAI API account.  
  - Edge cases: Same as CEO Agent’s LLM node.

---

#### 2.4 Merge & Document Composition

**Overview:**  
After receiving departmental outputs, the CEO Agent merges and reconciles them into a coherent Markdown plan and JSON payload. Then the workflow sets up metadata and creates a Google Doc from the Markdown.

**Nodes Involved:**  
- Configure metadata (Set node)  
- Create document file (HTTP Request to Google Drive API)

**Node Details:**

- **Configure metadata**  
  - Type: Set node  
  - Role: Prepares document metadata, including:  
    - Drive Folder ID (hardcoded)  
    - Document Content (Markdown plan from CEO Agent output)  
  - Input: CEO Agent output JSON  
  - Output: JSON with keys used by HTTP Request to create the document.  
  - Edge cases: Invalid folder ID or missing content would cause downstream errors.

- **Create document file**  
  - Type: HTTP Request node (Google Drive API multipart upload)  
  - Role: Creates a new Google Docs file containing the Markdown plan content.  
  - Configuration:  
    - Multipart request with JSON metadata and Markdown content in the body.  
    - Uses Google Drive OAuth2 credentials.  
    - Document named with company and timestamp.  
  - Input: Metadata and Markdown content from previous node.  
  - Output: Google Doc file metadata (including file ID).  
  - Edge cases: OAuth token expiration, API quota limits, invalid folder ID, content encoding issues.

---

#### 2.5 Document Export to PDF

**Overview:**  
Converts the created Google Doc into a PDF file for easy sharing.

**Nodes Involved:**  
- Convert to PDF (Google Drive Node)

**Node Details:**

- **Convert to PDF**  
  - Type: Google Drive node  
  - Role: Downloads the Google Doc file as PDF.  
  - Configuration:  
    - File ID taken from the created document.  
    - Operation set to download with conversion from Docs to PDF.  
  - Credentials: Google Drive OAuth2  
  - Input: Google Doc ID from previous node.  
  - Output: PDF binary data.  
  - Edge cases: File ID mismatch, API rate limits, conversion failures.

---

#### 2.6 Sharing (Slack Upload)

**Overview:**  
Uploads the generated PDF file to a Slack channel with a contextual message for stakeholder review.

**Nodes Involved:**  
- Upload a file (Slack node)

**Node Details:**

- **Upload a file**  
  - Type: Slack node  
  - Role: Uploads the PDF to a specified Slack channel with a message.  
  - Configuration:  
    - Channel ID hardcoded.  
    - Initial comment includes static text notifying the team about the Sales Season Plan.  
    - Uses OAuth2 credentials for Slack.  
  - Input: PDF file from Convert to PDF node.  
  - Output: Slack API response.  
  - Edge cases: Slack API rate limits, expired OAuth tokens, channel permission issues.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                       | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                          |
|----------------------------|----------------------------------|------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Initiates the workflow manually     | None                          | Edit Fields                 | Start the run (Manual Trigger) - Manual start keeps flow predictable and fast iteration.           |
| Edit Fields                | Set                              | Defines input brief for planning    | When clicking ‘Execute workflow’ | CEO Agent                   | Collect the brief - Captures company, products, audience, dates, channels, constraints, metrics.   |
| CEO Agent                  | LangChain AI Tool Agent          | Orchestrates multi-agent plan       | Edit Fields                   | Configure metadata, calls Marketing, Ops, Finance Agents | Orchestrate like a CEO - Reads brief, calls departments once, merges results into markdown+JSON.  |
| OpenAI Chat Model          | LangChain AI Language Model      | Provides LLM backend for CEO Agent  | CEO Agent (AI Tool)           | CEO Agent                   |                                                                                                    |
| Marketing Agent            | LangChain AI Tool Agent          | Produces marketing campaigns & calendar | CEO Agent (AI Tool)           | CEO Agent                   | Call Tool #1: Marketing Department - Max 3 campaigns, KPIs, realistic calendar.                    |
| OpenAI Chat Model1         | LangChain AI Language Model      | LLM backend for Marketing Agent     | Marketing Agent (AI Tool)     | Marketing Agent             |                                                                                                    |
| Operations Agent           | LangChain AI Tool Agent          | Prepares inventory, staffing, risks| CEO Agent (AI Tool)           | CEO Agent                   | Call Tool #2: Operations Department - Checklist with owners/due dates, highlight bottlenecks.     |
| OpenAI Chat Model2         | LangChain AI Language Model      | LLM backend for Operations Agent    | Operations Agent (AI Tool)    | Operations Agent            |                                                                                                    |
| Finance Agent              | LangChain AI Tool Agent          | Recommends pricing, budget, targets | CEO Agent (AI Tool)           | CEO Agent                   | Call Tool #3: Finance Department - Sensible budgets, tie discounts to campaigns & stock.          |
| OpenAI Chat Model3         | LangChain AI Language Model      | LLM backend for Finance Agent       | Finance Agent (AI Tool)       | Finance Agent               |                                                                                                    |
| Configure metadata         | Set                              | Sets Google Doc metadata & content  | CEO Agent                    | Create document file        | Set document metadata - Deterministic naming for easy filing/search.                              |
| Create document file       | HTTP Request (Google Drive API) | Uploads Markdown content as Google Doc | Configure metadata           | Convert to PDF              | Create the source document - Markdown-in, Doc-out to keep content editable.                        |
| Convert to PDF             | Google Drive                     | Exports Google Doc to PDF           | Create document file          | Upload a file               | Export to PDF - Save PDF filename matching doc title.                                             |
| Upload a file              | Slack                           | Shares PDF in Slack channel          | Convert to PDF                | None                       | Share the artifact - Include context in Slack message for clarity and review.                      |
| Sticky Note                | Sticky Note                     | Documentation & guidance            | None                         | None                       | Multi-agent architecture overview with video link and detailed explanation.                       |
| Sticky Note1               | Sticky Note                     | Documentation on Manual Trigger     | None                         | None                       | Explains importance of manual trigger as a start.                                                 |
| Sticky Note2               | Sticky Note                     | Documentation on Edit Fields        | None                         | None                       | Details inputs to collect and tips for brevity.                                                   |
| Sticky Note3               | Sticky Note                     | Documentation on CEO Agent          | None                         | None                       | Describes orchestration logic and output expectations.                                           |
| Sticky Note4               | Sticky Note                     | Documentation on Marketing Agent    | None                         | None                       | Explains Marketing Agent task and output JSON schema.                                            |
| Sticky Note5               | Sticky Note                     | Documentation on Operations Agent   | None                         | None                       | Explains Operations Agent task and output JSON schema.                                           |
| Sticky Note6               | Sticky Note                     | Documentation on Finance Agent      | None                         | None                       | Explains Finance Agent task and output JSON schema.                                              |
| Sticky Note7               | Sticky Note                     | Documentation on CEO merging outputs| None                         | None                       | Describes merging JSON outputs and producing markdown + JSON.                                   |
| Sticky Note8               | Sticky Note                     | Documentation on document creation  | None                         | None                       | Details metadata setup, doc creation, export to PDF steps.                                       |
| Sticky Note9               | Sticky Note                     | Documentation on Slack sharing      | None                         | None                       | Explains Slack upload message content and importance of context.                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed. This node starts the workflow on manual execution.

2. **Create Set Node for Inputs:**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Assign fields:  
     - Company: "AutomateWithMe"  
     - Product: "Automation Services"  
     - Sale window: "Aug to December annually"  
     - Primary channels: "Amazon, Ebay"  
     - Success metrics: "Revenue"  
     - Audience: "SME's owner (CEO, CFO, CIO)"  
   - Connect `When clicking ‘Execute workflow’` → `Edit Fields`.

3. **Add CEO Agent Node (LangChain AI Tool):**  
   - Name: `CEO Agent`  
   - Type: LangChain AI Tool Agent  
   - Configure system prompt with orchestration instructions:  
     - Define CEO role to call MarketingDept, OpsDept, FinanceDept exactly once.  
     - Include expected Markdown sections and JSON schema.  
     - Instructions to merge and resolve conflicts, prioritize feasibility.  
   - Text prompt dynamically injects input from `Edit Fields` (e.g., `Company`, `Product`, etc.).  
   - Connect `Edit Fields` → `CEO Agent`.

4. **Add OpenAI Chat Model Node for CEO Agent:**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain AI Language Model Node  
   - Credentials: Configure OpenAI API with your key.  
   - Model: Select GPT-4.1-mini or similar.  
   - Connect `OpenAI Chat Model` → `CEO Agent` (as AI languageModel).

5. **Add Marketing Agent Node (LangChain AI Tool Agent):**  
   - Name: `Marketing Agent`  
   - Type: LangChain AI Tool Agent  
   - Configure tool description with instructions to output max 3 campaigns, content calendar, notes in JSON format.  
   - Text prompt references input from CEO Agent.  
   - Connect `Marketing Agent` → `CEO Agent` (as AI tool).

6. **Add OpenAI Chat Model Node for Marketing Agent:**  
   - Name: `OpenAI Chat Model1`  
   - Type: LangChain AI Language Model Node  
   - Credentials: Same OpenAI account.  
   - Connect `OpenAI Chat Model1` → `Marketing Agent`.

7. **Add Operations Agent Node:**  
   - Name: `Operations Agent`  
   - Type: LangChain AI Tool Agent  
   - Configure tool description for inventory, staffing, fulfillment steps, risks output JSON.  
   - Connect `Operations Agent` → `CEO Agent` (AI tool).

8. **Add OpenAI Chat Model Node for Operations Agent:**  
   - Name: `OpenAI Chat Model2`  
   - Type: LangChain AI Language Model Node  
   - Credentials: Same OpenAI account.  
   - Connect `OpenAI Chat Model2` → `Operations Agent`.

9. **Add Finance Agent Node:**  
   - Name: `Finance Agent`  
   - Type: LangChain AI Tool Agent  
   - Configure tool description for discounts, budget split, targets output JSON.  
   - Connect `Finance Agent` → `CEO Agent` (AI tool).

10. **Add OpenAI Chat Model Node for Finance Agent:**  
    - Name: `OpenAI Chat Model3`  
    - Type: LangChain AI Language Model Node  
    - Credentials: Same OpenAI account.  
    - Connect `OpenAI Chat Model3` → `Finance Agent`.

11. **Add Set Node for Document Metadata:**  
    - Name: `Configure metadata`  
    - Type: Set  
    - Assignments:  
      - Drive Folder ID: Set to your Google Drive folder ID (e.g., "1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI")  
      - Document Content: Set to CEO Agent output Markdown (e.g., expression `={{ $json.output }}` or specific key path to Markdown text).  
    - Connect `CEO Agent` (main output) → `Configure metadata`.

12. **Add HTTP Request Node to Create Google Doc:**  
    - Name: `Create document file`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`  
    - Authentication: Google Drive OAuth2  
    - Body: Multipart/related content combining JSON metadata and Markdown content as text/markdown, with appropriate boundary strings.  
    - Connect `Configure metadata` → `Create document file`.

13. **Add Google Drive Node to Convert Doc to PDF:**  
    - Name: `Convert to PDF`  
    - Type: Google Drive  
    - Operation: Download file with conversion `docsToFormat` set to `application/pdf`  
    - File ID: Use the file ID returned by `Create document file` node.  
    - Connect `Create document file` → `Convert to PDF`.

14. **Add Slack Node to Upload PDF:**  
    - Name: `Upload a file`  
    - Type: Slack  
    - Resource: File  
    - Operation: Upload file with comment (e.g., "📄 The Sales Season Plan is ready! Please find the generated PDF file attached for review and next steps.")  
    - Channel ID: Set to your Slack channel ID  
    - Authentication: Slack OAuth2  
    - Connect `Convert to PDF` → `Upload a file`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                                            |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Multi-agent architecture template video: Watch a detailed walkthrough here [YouTube Video](https://www.youtube.com/watch?v=BfMY2jFJR9k)                                                                                                                                                       | Video tutorial for the multi-agent orchestration pattern in n8n.                                                                                          |
| The workflow is designed as a free bootstrap template for beginners to learn multi-agent orchestration with a clear company metaphor: CEO orchestrator and three specialist departments.                                                                                                      | Conceptual design guidance.                                                                                                                              |
| The CEO Agent merges the three departmental JSON outputs into both human-readable Markdown and a compact JSON for further automation (e.g., pushing next actions into PM tools like Jira or Asana).                                                                                             | Automation extension idea.                                                                                                                               |
| Credentials required: OpenAI API for all AI nodes, Google Drive OAuth2 for Docs creation and export, Slack OAuth2 for file upload.                                                                                                                                                             | Setup requirements.                                                                                                                                       |
| To customize: swap or add department agents, change output formats (DOCX, HTML), add approval steps (Slack send & wait), integrate grounding data sources (Sheets, DB), schedule with cron, add localization or guardrails (token limits).                                                    | Customization tips.                                                                                                                                       |
| Slack message always includes company and sales window context for clarity and audit trail.                                                                                                                                                                                                   | Best practice in sharing automated outputs.                                                                                                             |
| The workflow currently uses static input values in the Edit Fields node for demo; replace with dynamic form inputs or API triggers for production use.                                                                                                                                       | Scalability note.                                                                                                                                         |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive or protected elements. All manipulated data is legal and public.