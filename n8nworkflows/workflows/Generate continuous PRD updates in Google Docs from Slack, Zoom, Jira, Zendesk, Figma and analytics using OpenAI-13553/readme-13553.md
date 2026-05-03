Generate continuous PRD updates in Google Docs from Slack, Zoom, Jira, Zendesk, Figma and analytics using OpenAI

https://n8nworkflows.xyz/workflows/generate-continuous-prd-updates-in-google-docs-from-slack--zoom--jira--zendesk--figma-and-analytics-using-openai-13553


# Generate continuous PRD updates in Google Docs from Slack, Zoom, Jira, Zendesk, Figma and analytics using OpenAI

# Workflow Reference: Continuous PRD Automation

This document provides a technical breakdown of the n8n workflow designed to automate the continuous update of Product Requirement Documents (PRDs) in Google Docs. It aggregates signals from across a company’s stack—communications, project management, support, design, and analytics—and uses AI to synthesize these into structured product updates.

---

### 1. Workflow Overview

The workflow acts as a centralized "Product Intelligence" hub. It monitors multiple input channels for new feedback, meeting notes, or task updates, standardizes that data, and passes it to an AI Agent. This agent analyzes the context and appends a structured PRD update to a living Google Doc.

**Logical Blocks:**
*   **1.1 Data Ingestion (Layer 1):** Real-time triggers and scheduled fetches from Slack, Zoom, Jira, Zendesk, Figma, Customer Forms, and Webhooks.
*   **1.2 Data Standardization (Layer 2):** Transforming diverse JSON outputs into a uniform schema (source, timestamp, content).
*   **1.3 Signal Consolidation (Layer 3):** Merging multiple data streams into a single unified context for the AI.
*   **1.4 AI Synthesis (Layer 4):** Using GPT-4o-mini to extract feature requests, scope changes, and priorities into a structured JSON schema.
*   **1.5 Governance & Logging (Layer 5):** Formatting the AI output into a human-readable template and appending it to a Google Doc.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Transformation (Sources)
This block handles the unique API requirements for seven distinct sources.

*   **Slack Trigger:** Listens for `app_mention` or messages in a specific product channel.
*   **Zoom Recording & Transcription:** Fetches scheduled recordings and uses OpenAI’s Whisper model (`Transcribe Zoom Audio`) to convert audio to text.
*   **Jira/Zendesk Scheduled Fetch:** Polls Atlassian for comments and Zendesk for tickets tagged as "complaints" on an hourly interval.
*   **Figma Trigger:** Fires when a file is updated (monitors design feedback).
*   **Customer Form Trigger:** An internal n8n form for manual feedback entry.
*   **Platform Metadata (Webhook):** Receives raw analytics via POST request, followed by an HTTP Request node to fetch specific metrics.

**Transformation Nodes (Set Nodes):**
Every source has a corresponding "Format [Source] Data" node. These standardize fields like `source`, `timestamp`, and the primary text content (e.g., `comment_text`, `transcript`, or `metrics`) to ensure the AI receives consistent data structures.

#### 2.2 Signal Consolidation
*   **Nodes Involved:** `Merge All Data Sources`
*   **Overview:** A Merge node configured with 4+ inputs. It waits for the standardized objects from the previous layer to combine them into a single execution stream.
*   **Edge Case:** If one branch fails or provides no data, the workflow settings (`alwaysOutputData`) ensure the downstream AI process isn't necessarily blocked.

#### 2.3 AI Processing (The PRD Agent)
*   **Nodes Involved:** `PRD Analysis Agent`, `OpenAI Chat Model`, `Structured PRD Output`
*   **Overview:** This is the "brain" of the workflow. The Agent is provided with a System Message defining its persona as a "PRD Analysis Agent."
*   **Key Expressions:** It takes the stringified JSON from the Merge node: `{{ JSON.stringify($json, null, 2) }}`.
*   **Output Parser:** The `Structured PRD Output` node enforces a strict JSON schema including fields for `purpose`, `user_problem`, `acceptance_criteria`, and `out_of_scope`. This prevents "hallucinated" formatting.

#### 2.4 Document Export
*   **Nodes Involved:** `Update PRD Document` (Google Docs)
*   **Overview:** Takes the structured JSON from the AI and maps it to a Markdown-style text template.
*   **Configuration:** Uses the `insert` operation to append the update to the end of a specific Google Doc URL.
*   **Template Example:** `=== PRD UPDATE [Timestamp] === \n **Purpose:** {{ $json.output.purpose }} ...`

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Slack Trigger | Slack Trigger | Event Trigger | None | Format Slack Data | How it Works - Layer 1 |
| Customer Form Trigger | Form Trigger | Event Trigger | None | Format Form Data | How it Works - Layer 1 |
| Get Zoom Recordings | Zoom | Data Fetch | None | Transcribe Zoom Audio | How it Works - Layer 1 |
| Transcribe Zoom Audio | OpenAI | Audio Transcription | Get Zoom Recordings | Format Zoom Data | How it Works - Layer 1 |
| Schedule Jira Fetch | Schedule | Time Trigger | None | Get Jira Comments | How it Works - Layer 1 |
| Get Jira Comments | Jira | Data Fetch | Schedule Jira Fetch | Format Jira Data | How it Works - Layer 1 |
| Schedule Zendesk Fetch| Schedule | Time Trigger | None | Get Zendesk Tickets | How it Works - Layer 1 |
| Get Zendesk Tickets | Zendesk | Data Fetch | Schedule Zendesk Fetch | Format Zendesk Data | How it Works - Layer 1 |
| Figma Comment Trigger | Figma Trigger | Event Trigger | None | Format Figma Data | How it Works - Layer 1 |
| Platform Metdata | Webhook | Event Trigger | None | Fetch Platform Analytics | How it Works - Layer 1 |
| Fetch Platform Analytics| HTTP Request | Data Fetch | Platform Metdata | Format Platform Data | How it Works - Layer 1 |
| Format Nodes (All) | Set | Data Normalization | Respective Source Nodes | Merge All Data Sources | How it Works - Layer 2 |
| Merge All Data Sources | Merge | Data Consolidation | All Format Nodes | PRD Analysis Agent | How it Works - Layer 3 |
| PRD Analysis Agent | AI Agent | Contextual Analysis | Merge All Data Sources | Update PRD Document | How it Works - Layer 4 |
| OpenAI Chat Model | OpenAI Chat | AI Model Provider | Agent | Agent | GPT Setup |
| Structured PRD Output | Output Parser | Schema Enforcement | Agent | Agent | GPT Setup |
| Update PRD Document | Google Docs | Document Update | PRD Analysis Agent | None | How it Works - Layer 5 |

---

### 4. Reproducing the Workflow from Scratch

1.  **Set Up Triggers (Layer 1):**
    *   Create a **Slack Trigger** node: Choose "Message" and "App Mention" events.
    *   Create a **Zoom** node: Use the "Get All" operation for recordings. Connect it to an **OpenAI** node with the "Audio: Transcribe" operation.
    *   Create **Schedule Trigger** nodes for Jira and Zendesk (set to 1-hour intervals).
    *   Create a **Form Trigger** for internal feedback.
    *   Create a **Figma Trigger** node: Set the `Team ID` and trigger on `fileUpdate`.
2.  **Standardize Data (Layer 2):**
    *   For every trigger/fetch node, attach a **Set** node.
    *   Define a common schema in each: `source` (String), `timestamp` (Expression: `{{ $now }}`), and a `content` variable containing the main text from that source.
3.  **Consolidate (Layer 3):**
    *   Add a **Merge** node. Change the "Number of Inputs" to match your sources. Connect all Set nodes to it.
4.  **AI Intelligence (Layer 4):**
    *   Add an **AI Agent** node. Set the Prompt Type to "Define".
    *   Connect an **OpenAI Chat Model** node to the Agent's "Language Model" input (Model: `gpt-4o-mini`).
    *   Connect a **Structured Output Parser** to the Agent's "Output Parser" input. Define a JSON schema including `purpose`, `user_problem`, `user_value`, `target_users`, `user_feedback`, `assumptions`, `out_of_scope`, `acceptance_criteria`, and `next_steps`.
5.  **Finalize Output (Layer 5):**
    *   Add a **Google Docs** node. 
    *   Operation: **Update**. Action: **Insert Text**. 
    *   Paste the Markdown template using expressions to map the AI’s JSON output to the text (e.g., `{{ $json.output.purpose }}`).
6.  **Credentials:**
    *   Configure OAuth2 for Slack, Zoom, Google Docs, and Figma. 
    *   Configure API Tokens for Jira, Zendesk, and OpenAI.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Zoom Setup Guide** | [Video Tutorial](https://www.youtube.com/watch?v=BC6O_3LYgac) |
| **Jira API Token Guide** | [Video Tutorial](https://www.youtube.com/watch?v=T4z7lzqSZDY) |
| **Slack App Configuration** | [Video Tutorial](https://www.youtube.com/watch?v=qk5JH6ImK0I) |
| **Google Cloud Project Setup** | [Video Tutorial](https://www.youtube.com/watch?v=iieEHvu93dc) |
| **GPT System Prompt** | Customize the System Message in the AI Agent to define specific PRD styles (e.g., Amazon-style 6-pagers vs. Agile user stories). |
| **Analytics Placeholder** | The `Fetch Platform Analytics` node requires your specific internal API endpoint. |