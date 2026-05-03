Automated Compliance Control Scoring with CyberPulse, GPT-4o & Google Sheets

https://n8nworkflows.xyz/workflows/automated-compliance-control-scoring-with-cyberpulse--gpt-4o---google-sheets-9397


# Automated Compliance Control Scoring with CyberPulse, GPT-4o & Google Sheets

### 1. Workflow Overview

This workflow automates compliance control scoring and analysis by integrating CyberPulse's compliance scoring engine, GPT-4o AI summarization, and Google Sheets for input/output data management. It is designed for organizations needing to evaluate controls against multiple cybersecurity frameworks (e.g., ISO 27001, NIST CSF, SOC 2) by ingesting control responses, scoring compliance, generating AI-driven summaries and recommendations, and logging results in a structured sheet.

Logical blocks:

- **1.1 Input Reception & Preprocessing:** Load control data from Google Sheets and prepare fields for analysis.
- **1.2 CyberPulse Compliance Scoring:** Score and map each control with CyberPulse using control text, evidence, and notes.
- **1.3 AI Processing with GPT-4o:** Generate a JSON-formatted executive summary, findings, and recommendations from scored data.
- **1.4 Data Merging & Final Shaping:** Combine CyberPulse output with AI summary and normalize data fields.
- **1.5 Output Writing:** Append each processed control’s results into a Google Sheets results document.
- **1.6 Iteration Control:** Batch process controls one by one preserving context and ensuring scalable handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

- **Overview:**  
  This block fetches control records from a Google Sheet, normalizes and maps incoming data fields to consistent keys, and prepares them for scoring.

- **Nodes Involved:**  
  - Manual Trigger  
  - Get row(s) in sheet  
  - Edit Fields  

- **Node Details:**

  - **Manual Trigger**  
    - Type: Manual trigger node  
    - Role: Entry point to start workflow manually or for diagnostics, provides runId for traceability.  
    - Config: Default, no parameters.  
    - Inputs: None (trigger)  
    - Outputs: To "Get row(s) in sheet"  
    - Edge cases: None significant; manual start only.

  - **Get row(s) in sheet**  
    - Type: Google Sheets node (read operation)  
    - Role: Reads all rows from input Google Sheet containing control data.  
    - Config: Reads from Sheet1 of a specific Google Sheets document identified by ID. No value filtering or limits.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Edit Fields"  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Authentication failure, sheet not found, empty data.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Normalizes columns, maps input keys to consistent field names, trims text, and sets defaults to ensure data uniformity.  
    - Config: Sets fields like row_number, timestamp, control_description, response_text, implementation_notes, evidence URLs. Includes other fields as passthrough.  
    - Inputs: From "Get row(s) in sheet"  
    - Outputs: To "CyberPulse Compliance (Dev)"  
    - Edge cases: Missing or malformed fields; mitigated by defaults.

---

#### 2.2 CyberPulse Compliance Scoring

- **Overview:**  
  Scores and maps each control using CyberPulse’s API by combining control text, implementation notes, and up to four evidence URLs. Produces compliance scores, statuses, confidence, categories, and gap/action flags.

- **Nodes Involved:**  
  - CyberPulse Compliance (Dev)  

- **Node Details:**

  - **CyberPulse Compliance (Dev)**  
    - Type: Custom CyberPulse compliance node  
    - Role: Performs deterministic scoring and classification of controls, applying crosswalk mappings and evidence quality heuristics.  
    - Config:  
      - Control text concatenated from response_text + implementation_notes  
      - Crosswalk URL points to a JSON mapping for frameworks and clauses  
      - Up to 4 evidence URLs included  
    - Inputs: From "Edit Fields"  
    - Outputs: To "Loop Over Items"  
    - Credentials: CyberPulse HTTP Header Auth account  
    - Edge cases: API authentication failure, malformed input text, missing evidence URLs, network timeout.

---

#### 2.3 AI Processing with GPT-4o

- **Overview:**  
  For each control, generates a strict-JSON executive summary including an AI summary string, three findings, and three actionable recommendations, based on CyberPulse scoring and mapped frameworks.

- **Nodes Involved:**  
  - Loop Over Items  
  - Explain & Recommend  
  - Merge1  

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each control independently to enable isolated AI processing per item, preserving context and allowing scalable runs.  
    - Config: Default batch size, no filtering options.  
    - Inputs: From "CyberPulse Compliance (Dev)"  
    - Outputs:  
      - Main output: To "Explain & Recommend" (LLM)  
      - Secondary output: To "Merge1" (merging with original data)  
    - Edge cases: Empty input array, batch size issues.

  - **Explain & Recommend**  
    - Type: OpenAI GPT-4o Mini (LangChain node)  
    - Role: Calls GPT-4o model to generate AI summary, findings, and recommendations as a JSON object based on provided control data.  
    - Config:  
      - Model: gpt-4o-mini  
      - System prompt enforces strict JSON output with specified keys only  
      - Input message includes multiple fields from the control and scoring results, including rationale, categories, mappings, and rules for computing priority and flags  
      - Outputs JSON, parsed downstream  
    - Inputs: From "Loop Over Items"  
    - Outputs: To "Merge1"  
    - Credentials: OpenAI API key  
    - Edge cases: API rate limits, malformed response, JSON parsing failures, timeout.

  - **Merge1**  
    - Type: Merge node  
    - Role: Combines CyberPulse scoring/mapping results with AI-generated summaries by positional alignment, ensuring one unified data object per control.  
    - Config: Combine mode, include unpaired items, combine by position.  
    - Inputs:  
      - Primary: From "Explain & Recommend" (AI output)  
      - Secondary: From "Loop Over Items" (original CyberPulse output)  
    - Outputs: To "Parse + attach to each item"  
    - Edge cases: Mismatched array lengths causing unpaired data.

---

#### 2.4 Data Merging & Final Shaping

- **Overview:**  
  Final processing to normalize merged data into a flat structure suitable for reporting, cleaning up fields, ensuring consistent bullet formatting, and synthesizing missing summaries.

- **Nodes Involved:**  
  - Parse + attach to each item  

- **Node Details:**

  - **Parse + attach to each item**  
    - Type: Code node (JavaScript)  
    - Role:  
      - Parses merged JSON output from AI and CyberPulse nodes  
      - Normalizes text fields, trims, and removes leading bullet markers  
      - Converts AI findings and recommendations into uniformly formatted bulleted strings  
      - Computes fallback summaries if missing  
      - Ensures all fields are safely typed and present for output  
    - Config: Runs once per item, using helper functions for normalization and bullet formatting  
    - Inputs: From "Merge1"  
    - Outputs: To "Append row in sheet"  
    - Edge cases: Malformed AI JSON, missing fields, empty arrays.

---

#### 2.5 Output Writing

- **Overview:**  
  Appends each processed control’s fully prepared data as a new row into a results Google Sheet for downstream consumption and analysis.

- **Nodes Involved:**  
  - Append row in sheet  

- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets node (append operation)  
    - Role: Writes one row per control with all relevant fields including scores, AI outputs, mappings, evidence info, and timestamps.  
    - Config:  
      - Target sheet identified by document ID and sheet ID  
      - Explicit mapping of each column to dynamic expressions from JSON fields  
      - Does not convert field types; appends as strings mostly  
    - Inputs: From "Parse + attach to each item"  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: API authentication failure, sheet permission issues, quota limits, data format mismatches.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                            | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                               |
|---------------------------|--------------------------------------|--------------------------------------------|------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| Manual Trigger            | n8n-nodes-base.manualTrigger          | Start/diagnostics trigger                   | None                   | Get row(s) in sheet          | 🟢 Manual Trigger — Start/diagnostics; Receives POST and starts the run; echoes runId for tracing          |
| Get row(s) in sheet       | n8n-nodes-base.googleSheets           | Read input control rows from Google Sheet  | Manual Trigger         | Edit Fields                 | 🟩 Get row(s) in sheet — Read inputs; Loads model/routing settings from the config sheet                   |
| Edit Fields               | n8n-nodes-base.set                    | Normalize columns, map keys, set defaults  | Get row(s) in sheet    | CyberPulse Compliance (Dev) | 🟦 Edit Fields — Normalize columns; Maps incoming keys, trims text, and sets safe defaults                |
| CyberPulse Compliance (Dev)| n8n-nodes-cyberpulse-compliance-dev  | Score and map control compliance            | Edit Fields            | Loop Over Items             | 🟨 CyberPulse Compliance (Dev) — Score + map; Scores and maps each control using text and evidence        |
| Loop Over Items           | n8n-nodes-base.splitInBatches         | Iterate controls individually               | CyberPulse Compliance  | Explain & Recommend, Merge1 | 🟪 Loop Over Items — Iterate per control; Iterates each control independently for LLM → parse → append   |
| Explain & Recommend       | @n8n/n8n-nodes-langchain.openAi       | Generate AI summary, findings, recommendations | Loop Over Items      | Merge1                     | 🟦 Explain & Recommend (Message Model) — Exec summary; Generates strict-JSON AI executive summary          |
| Merge1                    | n8n-nodes-base.merge                   | Combine CyberPulse and AI outputs           | Explain & Recommend, Loop Over Items | Parse + attach to each item | 🟧 Merge1 — Combine model + original; Combines scoring/mapping with LLM summary by position                |
| Parse + attach to each item| n8n-nodes-base.code                    | Normalize and finalize row data for output  | Merge1                  | Append row in sheet          | 🟨 Parse + attach to each item — Final shaping; Merges CyberPulse and LLM outputs into a unified row       |
| Append row in sheet       | n8n-nodes-base.googleSheets           | Append processed rows to results Google Sheet | Parse + attach to each item | Loop Over Items             | 🟩 Append row in sheet — Write results; Appends one result row per item with core, scoring, AI fields      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - This will start the workflow manually.

2. **Create Google Sheets node “Get row(s) in sheet”**  
   - Type: Google Sheets (read)  
   - Credentials: Set up Google Sheets OAuth2 with access to your input sheet  
   - Configure to read from your input Google Sheet document and sheet (e.g., “Sheet1”)  
   - No filters; read all rows  
   - Connect output from Manual Trigger.

3. **Create Set node “Edit Fields”**  
   - Type: Set  
   - Use to map and normalize incoming fields: row_number, timestamp, control_description, response_text, implementation_notes, evidence_url_1 to evidence_url_4  
   - Enable “Include Other Fields” to pass through all other fields  
   - Connect from “Get row(s) in sheet”.

4. **Create CyberPulse Compliance node “CyberPulse Compliance (Dev)”**  
   - Type: CyberPulse compliance custom node  
   - Credentials: CyberPulse HTTP Header Auth with valid API credentials  
   - Parameters:  
     - Control Text: concatenate response_text and implementation_notes fields  
     - Crosswalk URL: set to the official crosswalk JSON URL for framework mappings  
     - Evidence URLs: map up to four evidence URL fields  
   - Connect from “Edit Fields”.

5. **Create SplitInBatches node “Loop Over Items”**  
   - Type: SplitInBatches  
   - Default batch size (e.g., 1) to process each control individually  
   - Connect from “CyberPulse Compliance (Dev)”.

6. **Create OpenAI node “Explain & Recommend”**  
   - Type: LangChain OpenAI node  
   - Credentials: OpenAI API key with access to GPT-4o-mini model  
   - Parameters:  
     - Model: gpt-4o-mini  
     - System prompt: instruct to return strict JSON with keys ai_summary, ai_findings, ai_recommendations  
     - User prompt: pass control fields and scoring results in JSON format, including rationale and rules for priority and flags  
     - Enable “JSON Output” for easier parsing  
   - Connect main output from “Loop Over Items”.

7. **Create Merge node “Merge1”**  
   - Type: Merge  
   - Mode: Combine, include unpaired, combine by position  
   - Inputs:  
     - Primary from “Explain & Recommend” (AI output)  
     - Secondary from “Loop Over Items” (original CyberPulse output)  
   - Connect output from “Explain & Recommend” and secondary output from “Loop Over Items”.

8. **Create Code node “Parse + attach to each item”**  
   - Type: Code (JavaScript)  
   - Mode: Run once per item  
   - Paste the normalization and parsing JS code that:  
     - Cleans bullet markers, converts AI arrays to bulleted strings, synthesizes missing summaries, and harmonizes fields  
   - Connect from “Merge1”.

9. **Create Google Sheets node “Append row in sheet”**  
   - Type: Google Sheets (append)  
   - Credentials: Google Sheets OAuth2 (same or different account with write access)  
   - Configure to append rows to the results Google Sheet and target sheet ID  
   - Map all output fields from the previous code node as columns in the sheet  
   - Connect from “Parse + attach to each item”.

10. **Finalize connections:**  
    - Connect “Append row in sheet” main output back to “Loop Over Items” input to continue batch processing until all items are done.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| CyberPulse Agent workflow automates evidence-aware control scoring (0–100) with deterministic gates and confidence.      | Workflow description sticky note explaining overall purpose and use case.                                                                             |
| Supports frameworks ISO 27001, NIST CSF, SOC 2, PCI DSS, Essential Eight, GDPR with tunable weights and thresholds.      | Workflow description sticky note.                                                                                                                    |
| Uses a public crosswalk JSON for framework mapping: https://gist.githubusercontent.com/gitadta/c6b7b69ae2a00f2a67e3bbac4b6648d4/raw/238ce80b0252702d4e6e9c19015bf958d0a0bad6/crosswalk.json | Referenced in CyberPulse node configuration.                                                                                                         |
| AI prompt enforces strict JSON output format for robust parsing downstream.                                              | Critical for avoiding parsing errors and ensuring consistent data structure from OpenAI node.                                                        |
| This workflow requires valid API credentials for Google Sheets, CyberPulse, and OpenAI to function end-to-end.           | Credential setup reminder.                                                                                                                            |

---

This documentation fully covers the workflow’s structure, logic, node configurations, data flows, and best practices for recreation or modification. It anticipates error scenarios such as API auth failures, empty inputs, and parsing issues, and explains how each block contributes to the overall compliance scoring pipeline.