Score B2B leads with OpenAI from webhook or form submissions

https://n8nworkflows.xyz/workflows/score-b2b-leads-with-openai-from-webhook-or-form-submissions-13763


# Score B2B leads with OpenAI from webhook or form submissions

### 1. Workflow Overview

The **Score B2B leads with OpenAI from webhook or form submissions** workflow is designed to automate the initial qualification process for inbound sales leads. It eliminates manual screening by using Artificial Intelligence (GPT-4o-mini) to evaluate lead data against a set threshold.

The workflow is structured into four functional phases:
1.  **Lead Reception & Configuration:** Captures lead data via an external webhook or manual trigger and defines the scoring threshold.
2.  **Data Normalization:** Ensures data consistency regardless of the source (live production data vs. manual test data).
3.  **AI Analysis:** Uses an OpenAI model to assign a numerical score (1–10) based on the lead's profile (name, company, email, website).
4.  **Routing & Outcome:** Categorizes leads as "Hot" (for immediate sales follow-up) or "Nurture" (for marketing sequences) based on the AI’s score.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry & Configuration
This block handles the entry points and sets the global logic parameters.
*   **Webhook Receive Lead:** (Webhook Node v2.1) Listens for POST requests at `/incoming-lead`. It returns the result of the last node in the workflow.
*   **Manual Test:** (Manual Trigger Node v1) Allows for on-demand testing within the n8n editor.
*   **Config:** (Edit Fields/Set Node v3.4) Sets a global variable `SCORE_THRESHOLD` (default: 7).
    *   *Edge Cases:* If the webhook body is empty, downstream nodes may fail during normalization.

#### 2.2 Source Routing & Normalization
This block identifies the data source and prepares a standard object for the AI.
*   **Route by Source:** (IF Node v2) Checks if `$json.body` exists. If true, it routes to normalization; if false (manual trigger), it routes to sample data.
*   **Normalize Lead Data:** (Edit Fields/Set Node v3.4) Maps webhook body fields (`name`, `email`, `company`, `website`) to a standard format.
*   **Sample Data:** (Edit Fields/Set Node v3.4) Provides hardcoded dummy data for testing the AI scoring logic.
    *   *Expressions:* Uses optional chaining (e.g., `{{ $json.body?.name ?? 'Lead' }}`) to prevent errors if fields are missing.

#### 2.3 AI Evaluation & Parsing
The core intelligence of the workflow.
*   **AI Score Lead:** (HTTP Request Node v4.2) Sends a JSON payload to the OpenAI API (`gpt-4o-mini`). It instructs the AI to return *only* a number between 1 and 10 based on lead fit.
    *   *Configuration:* Method: POST; URL: `https://api.openai.com/v1/chat/completions`; Temperature: 0.3.
    *   *Authentication:* Uses `openAiApi` predefined credentials.
*   **Parse Score:** (Code Node v2) A JavaScript snippet that extracts the numeric score from the AI response using regex.
    *   *Logic:* It ensures the score is an integer between 1 and 10. It also merges the original lead data back into the flow for context.

#### 2.4 Branching & Finalization
Determines the final action based on the score.
*   **Is Hot Lead?:** (IF Node v2) Compares the parsed score against the `SCORE_THRESHOLD`.
*   **Hot Lead / Nurture Lead:** (Edit Fields/Set Nodes v3.4) Appends status tags (`hot` or `nurture`) and suggested actions (`immediate_followup` or `add_to_sequence`).
*   **Merge Output:** (Merge Node v3) Unifies the two branches into a single output stream for delivery to a CRM or notification system.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook Receive Lead | Webhook | Production Trigger | None | Config | **Step 1 — Triggers** • Webhook: POST to `/incoming-lead` • Manual: Test with sample data |
| Manual Test | Manual Trigger | Testing Trigger | None | Config | **Step 1 — Triggers** • Webhook: POST to `/incoming-lead` • Manual: Test with sample data |
| Config | Set (Edit Fields) | Parameter Setup | Webhook, Manual Test | Route by Source | |
| Route by Source | IF | Logic Branching | Config | Normalize Lead Data, Sample Data | |
| Normalize Lead Data | Set (Edit Fields) | Data Standardizing | Route by Source | AI Score Lead | **Step 2–4 — Process** Normalize → AI Score (OpenAI) → Parse |
| Sample Data | Set (Edit Fields) | Test Data Injection | Route by Source | AI Score Lead | **Step 2–4 — Process** Normalize → AI Score (OpenAI) → Parse |
| AI Score Lead | HTTP Request | AI Processing | Normalize / Sample | Parse Score | **Step 2–4 — Process** Normalize → AI Score (OpenAI) → Parse |
| Parse Score | Code | Data Extraction | AI Score Lead | Is Hot Lead? | **Step 2–4 — Process** Normalize → AI Score (OpenAI) → Parse |
| Is Hot Lead? | IF | Qualification | Parse Score | Hot Lead, Nurture Lead | **Step 5 — Route** Hot (≥7) → immediate follow-up | Nurture → add to sequence |
| Hot Lead | Set (Edit Fields) | Metadata Tagging | Is Hot Lead? | Merge Output | **Step 5 — Route** Hot (≥7) → immediate follow-up | Nurture → add to sequence |
| Nurture Lead | Set (Edit Fields) | Metadata Tagging | Is Hot Lead? | Merge Output | **Step 5 — Route** Hot (≥7) → immediate follow-up | Nurture → add to sequence |
| Merge Output | Merge | Result Consolidation | Hot Lead, Nurture Lead | None | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Triggers:**
    *   Create a **Webhook** node. Set HTTP Method to `POST` and path to `incoming-lead`.
    *   Create a **Manual Trigger** node.
2.  **Configuration:**
    *   Add an **Edit Fields (Set)** node named "Config". Create a Number variable `SCORE_THRESHOLD` and set it to `7`. Connect both triggers to this node.
3.  **Source Filtering:**
    *   Add an **IF** node ("Route by Source"). Condition: `Boolean` -> `$json.body` is not undefined.
    *   Connect the **True** output to a new **Edit Fields** node ("Normalize Lead Data"). Map `name`, `email`, `company`, and `website` using the expression `{{ $json.body.field_name ?? '' }}`.
    *   Connect the **False** output to a new **Edit Fields** node ("Sample Data"). Manually enter strings for `name`, `email`, `company`, and `website`.
4.  **AI Integration:**
    *   Add an **HTTP Request** node ("AI Score Lead").
    *   URL: `https://api.openai.com/v1/chat/completions`. Method: `POST`.
    *   Authentication: Select `Predefined Credential Type` and choose `OpenAI API`.
    *   Body: Use JSON. Set `model` to `gpt-4o-mini`. In `messages`, set the user prompt: *"Score this B2B lead from 1-10. Reply with ONLY the number. Name: {{ $json.name }} Email: {{ $json.email }} Company: {{ $json.company }} Website: {{ $json.website }}"*.
5.  **Post-Processing:**
    *   Add a **Code** node ("Parse Score"). Use JavaScript to extract the AI's response text, regex it for a digit, and merge it with the original input data.
6.  **Routing:**
    *   Add an **IF** node ("Is Hot Lead?"). Condition: `Number` -> `score` >= `SCORE_THRESHOLD`.
    *   Create two **Edit Fields** nodes: "Hot Lead" (set `status` to `hot`) and "Nurture Lead" (set `status` to `nurture`).
    *   Connect them to a **Merge** node to unify the output.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Model Selection** | This workflow uses `gpt-4o-mini` for cost-efficiency. For more complex scoring logic, consider `gpt-4o`. |
| **Response Mode** | The Webhook is set to `lastNode`, meaning the final merged JSON (lead + score + status) is returned to the requester. |
| **OpenAI Credentials** | Requires an active OpenAI API Key configured in n8n's credential manager. |
| **Scoring Logic** | The AI prompt is "Zero-Shot". For better accuracy, include examples of "good" vs "bad" leads in the prompt. |