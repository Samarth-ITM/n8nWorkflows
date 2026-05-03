Generate documents from saved templates and variables using Autype

https://n8nworkflows.xyz/workflows/generate-documents-from-saved-templates-and-variables-using-autype-13784


# Generate documents from saved templates and variables using Autype

This document provides a technical overview and configuration guide for the n8n workflow: **Render a Saved Autype Document with Variable Overrides**.

### 1. Workflow Overview
The purpose of this workflow is to automate the generation of personalized PDF documents (specifically Employee of the Month certificates) using the Autype platform. It demonstrates how to take user input from a form, transform that data into complex types (like tables and images), and inject them into a pre-defined Autype template.

The workflow is divided into two main logical paths:
*   **Setup Path:** A one-time initialization sequence to create a project and a document template in Autype.
*   **Production Path:** A user-facing trigger that collects data via an n8n Form and renders a PDF document with dynamic variable overrides.

---

### 2. Block-by-Block Analysis

#### 2.1 One-Time Setup (Project & Template Creation)
This block is used to initialize the environment in Autype. It creates the necessary folder structure and the JSON-based document template containing placeholders.

*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Create Project`, `Create Document`.
*   **Node Details:**
    *   **When clicking ‘Execute workflow’**: Manual trigger to start the setup.
    *   **Create Project (Autype Node)**: 
        *   **Role:** Creates a container named "Certificates" in Autype.
        *   **Configuration:** Resource: `document`, Operation: `createProject`.
    *   **Create Document (Autype Node)**:
        *   **Role:** Defines the visual structure and default variables of the certificate.
        *   **Key Expressions:** Uses `{{ $json.id }}` from the previous node as the `projectId`.
        *   **Configuration:** Includes a large JSON object in the `content` field defining styles (H1, H2, Tables), headers (Logo), and sections (Flow with spacers, text, and variable references like `{{recipientName}}`).

#### 2.2 Input Reception & Data Mapping
This block captures the specific details for an individual certificate and prepares the data format required by the Autype API.

*   **Nodes Involved:** `Certificate Form`, `Set Form Variables`.
*   **Node Details:**
    *   **Certificate Form (Form Trigger)**:
        *   **Role:** Captures Recipient Name, ID, Month, Achievements, and Image URLs.
        *   **Edge Cases:** "Achievements" requires a specific "key:value" format for later parsing.
    *   **Set Form Variables (Set Node)**:
        *   **Role:** Normalizes form inputs and provides fallback values for images.
        *   **Key Expressions:** 
            *   `profilePhotoUrl`: `{{ $json['Profile Photo URL'] || 'https://placehold.co/...' }}`
            *   `documentId`: Hardcoded ID obtained from the Setup Path.

#### 2.3 Document Rendering
The final block synchronizes the template with the live data to produce the binary PDF file.

*   **Nodes Involved:** `Get Document Variables`, `Render Document with Variables`.
*   **Node Details:**
    *   **Get Document Variables (Autype Node)**:
        *   **Role:** (Optional) Fetches the schema of the document to ensure variables match.
    *   **Render Document with Variables (Autype Node)**:
        *   **Role:** Performs the heavy lifting of data transformation and PDF generation.
        *   **Complex Mapping:** 
            *   **Tables:** The `achievementsRaw` string is split by commas, then by colons, and mapped into a JSON array structure for the Autype "table" type.
            *   **Images:** Formats URLs into Autype "image" objects with defined widths and alignments.
        *   **Configuration:** `downloadOutput: true` ensures the node returns the binary PDF file.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When clicking ‘Execute workflow’** | Manual Trigger | Initializer | None | Create Project | |
| **Create Project** | Autype | Project Creation | Execute workflow | Create Document | ONE-TIME SETUP — Run once, then disable |
| **Create Document** | Autype | Template Definition | Create Project | None | ONE-TIME SETUP — Run once, then disable |
| **Certificate Form** | Form Trigger | User Interface | None | Set Form Variables | Collects: recipient name, employee ID, award month, achievements... |
| **Set Form Variables** | Set | Data Normalization | Certificate Form | Get Document Variables | |
| **Get Document Variables** | Autype | Schema Verification | Set Form Variables | Render Document | Fetches the variable schema from the saved document |
| **Render Document with Variables** | Autype | PDF Generation | Get Document Variables | None | Injects: `recipientName`, `EmployeeID`, `awardMonth`, `achivements`, `profilePhoto`... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Install Community Node:** Go to Settings -> Community Nodes and install `n8n-nodes-autype`.
2.  **Credentials:** Create an **Autype API** credential using your API key from [app.autype.com](https://app.autype.com).
3.  **Setup Logic:**
    *   Create a **Manual Trigger** connected to an **Autype Node** (Action: Create Project).
    *   Connect another **Autype Node** (Action: Create Document). Set `Project ID` to the ID from the previous step. Paste the template JSON into the `Content` field.
    *   **Execute once**, then copy the resulting `document ID`.
4.  **Production Logic:**
    *   Create an **n8n Form Trigger**. Add fields for: Recipient Name (String), Employee ID (String), Award Month (String), Achievements (Textarea), and Image URLs.
    *   Add a **Set Node**. Map the form fields to clean variable names. **Crucial:** Paste the `document ID` from Step 3 into a field named `documentId`.
    *   Add an **Autype Node** (Action: Render Document).
        *   Set `Document ID` via expression: `{{ $json.documentId }}`.
        *   Enable `Download Output`.
        *   In `Variables`, use a JSON expression to map the Set node values. For the **Achievements Table**, use a `.split().map()` function to convert the comma-separated string into a nested array.
5.  **Connection:** Link the Form Trigger -> Set Node -> Autype Render node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **API Keys Management** | [Autype API Documentation](https://docs.autype.com/getting-started/editor/settings#api-keys) |
| **Platform Access** | [Autype Web Editor](https://app.autype.com) |
| **Prerequisite** | This workflow requires a self-hosted n8n instance to install community nodes. |
| **Variable Types** | Autype supports String, Number, Table (array of arrays), and Image (object with src). |