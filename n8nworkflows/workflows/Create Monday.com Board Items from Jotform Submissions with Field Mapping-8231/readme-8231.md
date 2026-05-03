Create Monday.com Board Items from Jotform Submissions with Field Mapping

https://n8nworkflows.xyz/workflows/create-monday-com-board-items-from-jotform-submissions-with-field-mapping-8231


# Create Monday.com Board Items from Jotform Submissions with Field Mapping

### 1. Workflow Overview

This workflow automates the process of capturing new form submissions from Jotform and creating corresponding items on a Monday.com board with mapped columns. It is designed primarily for managing leads, requests, or intake forms by transferring structured data such as emails, dates, dropdown selections, instructions, and derived fields (like domain extracted from email) directly into a Monday.com project board.

The workflow is organized into two main logical blocks:

- **1.1 Input Reception:** Receiving and triggering on new Jotform submissions.
- **1.2 Data Mapping and Monday.com Item Creation:** Mapping the Jotform submission fields to Monday.com board columns and creating new board items accordingly.

Supporting setup notes and instructions are included as sticky notes throughout the workflow for credential setup and configuration guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block listens for new submissions on a specified Jotform form and acts as the trigger for the workflow.
- **Nodes Involved:**  
  - `Jotform Form`

- **Node Details:**

  - **Node Name:** Jotform Form  
    - **Type and Role:** JotForm Trigger node; triggers workflow on new submissions to a specific Jotform form.  
    - **Configuration:**  
      - Connected to a form with ID `252445218058053` (selected from the dropdown).  
      - Uses a webhook internally to receive real-time submission data.  
    - **Expressions/Variables:**  
      - Emits full form submission data including fields like Name (with subfields first and last), Email, Start Date (structured as year, month, day), Engagement Type, Campaign Type, and Instructions.  
    - **Input/Output:**  
      - No input connections (trigger node).  
      - Output connected to `Monday.com Post` node.  
    - **Version-specific Requirements:** Compatible with n8n v1.0+; webhook must be accessible externally.  
    - **Potential Failures:**  
      - Webhook failure due to network or permission issues.  
      - Missing or changed form ID causing no triggers.  
      - API key misconfiguration leading to authentication errors.

#### 1.2 Data Mapping and Monday.com Item Creation

- **Overview:** This block transforms the incoming Jotform submission data into the correct format and field mapping required by Monday.com, then creates a new item on the specified board and group.  
- **Nodes Involved:**  
  - `Monday.com Post`

- **Node Details:**

  - **Node Name:** Monday.com Post  
    - **Type and Role:** Monday.com node; creates a new board item with mapped column values.  
    - **Configuration:**  
      - Board ID set to `9954397625`.  
      - Group ID set to `topics`.  
      - Resource set to `boardItem` with action to create new item.  
      - Item `name` set dynamically by concatenating the first and last names from submission (`{{$json.Name.first}} {{$json.Name.last}}`).  
      - `columnValues` field is a JSON string constructed by a JavaScript function that maps several form fields to their corresponding Monday.com column IDs:  
        - Email to a text column (`text_mkvdj8v3`)  
        - Start Date formatted as YYYY-MM-DD to a date column (`date_mkvdg4aa`)  
        - Engagement Type mapped via label-to-ID dictionary to a dropdown column (`dropdown_mkvdjwra`)  
        - Campaign Type mapped similarly to a dropdown column (`dropdown_mkvdd9v3`)  
        - Campaign Type also stored as text label (`text_mkvd2md9`)  
        - Instructions to a text column (`text_mkvd1bj2`)  
        - Domain extracted from email and stored as text (`text_mkvd5w3y`)  
      - Uses credential named "Monday.com account" for API authentication.  
    - **Expressions/Variables:**  
      - Uses inline JavaScript in the `columnValues` parameter to build JSON with conditional checks and mappings.  
    - **Input/Output:**  
      - Input from `Jotform Form` node.  
      - No output nodes (end of workflow).  
    - **Version-specific Requirements:** Requires Monday.com API token with write permissions; n8n Monday.com node v1+.  
    - **Potential Failures:**  
      - API authentication failures due to invalid or expired token.  
      - Incorrect board, group, or column IDs causing errors.  
      - Mismatched dropdown label-to-ID mappings leading to missing dropdown selections.  
      - Date formatting errors if date fields are missing or malformed.  
      - JSON stringification failures if unexpected data structure is encountered.

#### Supporting Setup and Documentation Notes (Sticky Notes)

- **Sticky Note2:**  
  - Provides a general overview of the workflow’s purpose and use cases.  
- **Sticky Note11:**  
  - Detailed Monday.com setup instructions including credential creation, board/group/column ID identification, and label-to-ID mapping for dropdowns.  
- **Sticky Note64:**  
  - Simple instructions for setting up Jotform API key and selecting form in n8n.  
- **Sticky Note65:**  
  - Detailed Monday.com setup guide reiterating the steps and confirming the importance of correct IDs and token.

---

### 3. Summary Table

| Node Name       | Node Type               | Functional Role                          | Input Node(s)     | Output Node(s)     | Sticky Note                                                                                             |
|-----------------|-------------------------|----------------------------------------|-------------------|--------------------|-------------------------------------------------------------------------------------------------------|
| Jotform Form    | JotForm Trigger         | Trigger on new Jotform submissions     | -                 | Monday.com Post    | See Sticky Note64 for Jotform setup instructions.                                                     |
| Monday.com Post | Monday.com              | Create new item on Monday.com board    | Jotform Form      | -                  | See Sticky Note11 and Sticky Note65 for Monday.com setup and credential instructions.                 |
| Sticky Note2    | Sticky Note             | Overview of workflow purpose            | -                 | -                  | Describes workflow use for routing Jotform submissions to Monday.com board items.                     |
| Sticky Note11   | Sticky Note             | Monday.com setup instructions           | -                 | -                  | Detailed Monday.com API token, board/group/column ID setup, and dropdown value mapping guide.         |
| Sticky Note64   | Sticky Note             | Jotform setup instructions              | -                 | -                  | Simple setup for Jotform API key and form selection.                                                  |
| Sticky Note65   | Sticky Note             | Additional Monday.com setup instructions| -                 | -                  | Reinforces Monday.com API token creation and board configuration steps.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Add a new node and select `JotForm Trigger`.  
   - In the node parameters, select or enter the Jotform form ID you want to listen to (e.g., `252445218058053`).  
   - Ensure your Jotform API key is configured in n8n credentials for the Jotform node to authenticate.  
   - Save the node (this will create a webhook in n8n to receive submissions).

2. **Create Monday.com Node to Create Board Item**  
   - Add a new node and select `Monday.com`.  
   - Configure credentials by creating a new Monday.com credential with your personal API token generated from Monday.com Admin/Developers section.  
   - Set the node parameters:  
     - `Resource`: `boardItem`  
     - `Operation`: `Create`  
     - `Board ID`: Enter your Monday.com board ID (e.g., `9954397625`).  
     - `Group ID`: Enter the target group ID (e.g., `topics`).  
     - `Name`: Use expression to concatenate first and last name from Jotform submission: `={{ $json.Name.first }} {{ $json.Name.last }}`.  
     - `Additional Fields` → `Column Values`: Use an expression with inline JavaScript to build a JSON string mapping form fields to Monday.com columns as follows:  
       ```javascript
       (() => {
         const out = {};
         out.text_mkvdj8v3 = $json.Email || "";
         if ($json['Start Date']) {
           const y = String($json['Start Date'].year || '');
           const m = String($json['Start Date'].month || '').padStart(2,'0');
           const d = String($json['Start Date'].day || '').padStart(2,'0');
           out.date_mkvdg4aa = { date: (y && m && d) ? `${y}-${m}-${d}` : '' };
         }
         const engMap = { "Engagement A": 1, "Engagement B": 2 };
         const eng = $json['Engagement Type'];
         if (eng && engMap[eng]) out.dropdown_mkvdjwra = { ids: [engMap[eng]] };
         const campMap = { "Campaign A": 1, "Campaign B": 2 };
         const camp = $json['Campaign Type'];
         if (camp && campMap[camp]) out.dropdown_mkvdd9v3 = { ids: [campMap[camp]] };
         out.text_mkvd2md9 = $json['Campaign Type'] || "";
         out.text_mkvd1bj2 = $json.Instructions || "";
         out.text_mkvd5w3y = ($json.Email && $json.Email.includes("@")) ? $json.Email.split("@")[1] : "";
         return JSON.stringify(out);
       })()
       ```
   - Connect the output of the `JotForm Trigger` node as input to this node.

3. **Connect Nodes**  
   - Connect the `Jotform Form` node output to the `Monday.com Post` node input.  
   - This ensures the workflow triggers on each new submission and creates a Monday.com item accordingly.

4. **Credential Setup**  
   - For Jotform: Add your API key under Credentials → Jotform API Key.  
   - For Monday.com: Create a new credential by pasting your generated API token.

5. **Test Workflow**  
   - Submit a test form entry in Jotform to trigger the workflow.  
   - Verify that an item is created on the Monday.com board with the correct mapped fields.

6. **Optional: Add Sticky Notes**  
   - Add sticky notes with setup instructions and helpful information for future maintainers or users, similar to the ones present in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                               | Context or Link                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| This workflow enables routing of leads, requests, or intake form submissions directly into Monday.com for streamlined team management.                                                                                     | Workflow overview (Sticky Note2)                                             |
| Monday.com API tokens are generated via Admin/Developers → API section in Monday.com and must have appropriate permissions to create and edit board items.                                                                 | Monday.com setup (Sticky Note11 & Sticky Note65)                            |
| Jotform API keys are created via Jotform Account Settings → API. The form ID is required to set up the trigger in n8n.                                                                                                      | Jotform setup (Sticky Note64)                                                |
| Dropdown mappings from labels to Monday.com internal IDs must be customized per your board configuration; examples provided map "Engagement A" → 1, "Engagement B" → 2, etc. Replace with your actual IDs.                    | Label to ID mapping in Monday.com Post node                                 |
| Contact info for workflow customization or support: Robert Breen, rbreen@ynteractive.com, https://www.linkedin.com/in/robert-breen-29429625/, https://ynteractive.com                                                        | Provided in Sticky Note11                                                    |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected content. All handled data is legal and public.