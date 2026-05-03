Create manager tasks from Keephub form submissions with OpenAI GPT-4.1

https://n8nworkflows.xyz/workflows/create-manager-tasks-from-keephub-form-submissions-with-openai-gpt-4-1-13828


# Create manager tasks from Keephub form submissions with OpenAI GPT-4.1

This document provides a technical analysis of the n8n workflow designed to automate the creation of manager tasks following Keephub form submissions.

### 1. Workflow Overview
The **"Create manager tasks from Keephub form submissions with OpenAI GPT-4.1"** workflow acts as an intelligent bridge between employee feedback and management action. When a form is submitted in Keephub, the workflow extracts the data, identifies the appropriate manager in the organizational hierarchy, uses AI to design a context-specific follow-up task, and assigns it automatically.

**Logical Blocks:**
*   **1.1 Webhook & Filtering:** Captures incoming form data and filters out non-relevant events (edits or status changes).
*   **1.2 Context Resolution:** Normalizes form fields, identifies the submitter, and determines the target organizational unit (Manager or Root).
*   **1.3 AI Task Design:** Generates a dynamic prompt, invokes GPT-4.1 to draft the task structure, and validates the output.
*   **1.4 Task Execution:** Formulates the final Keephub API payload and creates the task in the system.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook & Filtering
This block ensures the workflow only triggers on fresh submissions to prevent duplicate tasks or loops.
*   **Nodes:** `Keephub Form Webhook`, `Status Change?`, `Is Edit?`.
*   **Logic:** The workflow proceeds only if `statusChanged` is false and `isEdited` is false.
*   **Edge Cases:** If a user submits a form and then immediately edits it, the edit will be ignored by this specific logic flow.

#### 2.2 Context Resolution
Resolves "who" submitted the form and "where" the follow-up task should be sent.
*   **Nodes:** `Extract Form Data`, `Get Submitter`, `Resolve Org Node`, `Get Parent Node`, `Get Root Node`.
*   **Technical Role:** 
    *   `Extract Form Data` uses JavaScript to normalize various Keephub field types (e.g., converting `NodeSelector` objects to simple IDs).
    *   `Resolve Org Node` prioritizes a `NodeSelector` field from the form; if absent, it defaults to the submitter's primary organizational unit.
    *   `Get Parent Node` attempts to find the manager's node. If the submitter is already at the top of the hierarchy, the "Error" path triggers `Get Root Node` to ensure the task has a destination.
*   **Potential Failure:** Auth errors on `Get Submitter` if credentials expire.

#### 2.3 AI Task Design
The core intelligence engine that transforms raw data into a structured task.
*   **Nodes:** `Get Form Schema`, `⚙️ Config`, `Build AI Prompt`, `Design Task with AI`.
*   **Configuration:** 
    *   `⚙️ Config` allows setting a `groupId` (to target specific manager groups) and `taskDueDaysFromNow`.
    *   `Build AI Prompt` dynamically maps submitted values back to their human-readable labels from the schema.
*   **AI Model:** GPT-4.1.
*   **Instructions:** The prompt forces the AI to detect the language (English/Dutch), reference specific submission data in the title, and include 3–6 actionable fields (RadioButtons, DatePickers, etc.).

#### 2.4 Task Composition & Creation
Finalizes the task structure and interacts with the Keephub API.
*   **Nodes:** `Compose Task`, `Create Task`.
*   **Technical Role:** 
    *   `Compose Task` performs heavy lifting: it parses AI JSON, generates unique IDs for new form fields using `crypto`, calculates due dates using Luxon, and builds a deep-link URL back to the original submission.
    *   `Create Task` sends a POST request to the Keephub `task` resource.
*   **Validation:** If the AI fails to provide a title or intro text, `Compose Task` throws an error to prevent malformed tasks.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Keephub Form Webhook | Webhook | Entry Point | - | Status Change? | - |
| Status Change? | If | Event Filter | Keephub Form Webhook | Is Edit? | Webhook Filter: Ensures only NEW form submissions trigger task creation. |
| Is Edit? | If | Event Filter | Status Change? | Extract Form Data | Webhook Filter: Filters out status changes and edits. |
| Extract Form Data | Code | Normalization | Is Edit? | Get Submitter | Normalizes all field types into a clean, indexed structure. |
| Get Submitter | Keephub | User Identity | Extract Form Data | Resolve Org Node | Fetches the employee's profile and memberships. |
| Resolve Org Node | Code | Targeting | Get Submitter | Get Parent Node | Determines which org node the submission belongs to. |
| Get Parent Node | Keephub | Hierarchy Lookup | Resolve Org Node | Extract Parent Target, Get Root Node | Finds the manager one level up in the hierarchy. |
| Get Root Node | Keephub | Fallback Lookup | Get Parent Node | Extract Root Target | Fetches the actual organizational root to prevent errors. |
| Extract Parent Target | Code | Path Resolver | Get Parent Node | Get Form Schema | Extracts ID from parent node. |
| Extract Root Target | Code | Path Resolver | Get Root Node | Get Form Schema | Extracts ID from root node. |
| Get Form Schema | Keephub | Schema Lookup | Extract Parent Target, Extract Root Target | ⚙️ Config | Retrieves field labels for building AI prompts. |
| ⚙️ Config | Set | User Config | Get Form Schema | Build AI Prompt | Edit values like groupId and taskDueDaysFromNow here. |
| Build AI Prompt | Code | Prompt Engineering | ⚙️ Config | Design Task with AI | Creates a structured prompt with form data and rules. |
| Design Task with AI | OpenAI | AI Reasoning | Build AI Prompt | Compose Task | GPT-4.1 analyzes the submission and designs the task. |
| Compose Task | Code | Payload Builder | Design Task with AI | Create Task | Validates AI output and builds full Keephub API payload. |
| Create Task | Keephub | API Execution | Compose Task | - | Posts the final task to the Keephub API. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:** 
    *   Create a **Webhook Node** (POST) with the path `keephub-form-to-task`.
    *   Add two **If Nodes** to filter: `{{ $json.body.data.statusChanged }} == false` and `{{ $json.body.data.isEdited }} == false`.
2.  **Data Extraction:**
    *   Add a **Code Node** to map `body.data.values`. Use logic to handle `NodeSelector`, `RadioButtons`, and `CheckboxGroup` types.
3.  **Hierarchy Resolution:**
    *   Add a **Keephub Node** (`user:get`) using the `updatedBy` ID from the webhook.
    *   Add a **Code Node** to find the target `orgNodeId` (Check form fields for a NodeSelector first, then user's `orgunits[0]`).
    *   Add a **Keephub Node** (`orgchart:getParent`). **Crucial:** Set "On Error" to "Continue (using error output)".
    *   Connect the error output of `getParent` to a **Keephub Node** (`orgchart:getAncestors`) to find the Root node as a fallback.
4.  **Schema and Config:**
    *   Add a **Keephub Node** (`content:getById`) using the `contentRef` from the webhook.
    *   Add a **Set Node** for configuration constants: `groupId` (String), `taskDueDaysFromNow` (Number), and `workflow2WebhookUrl` (String).
5.  **AI Implementation:**
    *   Add a **Code Node** to construct the prompt. It must include the form title, a list of submitted fields/values, and JSON formatting instructions for the AI.
    *   Add an **OpenAI Node** using the `gpt-4.1` model. Pass the prompt from the previous node.
6.  **Payload Composition:**
    *   Add a **Code Node** to:
        *   Parse the AI JSON response.
        *   Generate unique IDs for fields (use `crypto.randomBytes(12).toString('hex')`).
        *   Build the `orgchartSelection` object using the resolved Parent/Root ID.
        *   Construct the `template.form.fields` array.
7.  **Final Step:**
    *   Add a **Keephub Node** (`task:create`) and map the JSON body from the Compose node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Keephub API Documentation** | Use for verifying field element types like `Smileys`, `Stars`, and `NodeSelector`. |
| **Group Targeting Requirement** | To target specific roles, find the 24-character hex ID in the Keephub Admin Panel > Groups. |
| **AI Model Version** | Optimized for GPT-4.1; lower models may struggle with strict JSON schema adherence. |
| **Deep Linking Logic** | The task automatically generates a link to `/system/content/{contentRef}/formValueId/{id}/view`. |

***

*Disclaimer: This documentation is generated based on the provided n8n workflow JSON and is intended for technical integration purposes.*