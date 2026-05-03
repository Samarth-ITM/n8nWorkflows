Extract and Structure X Post Comments with Airtop Browser Automation

https://n8nworkflows.xyz/workflows/extract-and-structure-x-post-comments-with-airtop-browser-automation-4208


# Extract and Structure X Post Comments with Airtop Browser Automation

---
### 1. Workflow Overview

This workflow automates the extraction and structuring of comments from a specified post on X (formerly Twitter) using Airtop browser automation. It is designed for marketing professionals and analysts who need to gather and analyze user comments efficiently without manual scraping.

The workflow logically divides into these blocks:

- **1.1 Input Reception:** Collects inputs either from a user-submitted form or from another workflow invocation.
- **1.2 Parameter Normalization:** Standardizes input fields to consistent variable names for downstream use.
- **1.3 Comment Extraction via Airtop:** Initiates a new Airtop browser session, navigates to the specified X post URL, and extracts up to the maximum number of comments with specified details.
- **1.4 Data Formatting:** Refines and structures the extracted data for further use or export.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block captures the essential input parameters through two possible entry points: a user form submission or execution triggered by another workflow.
- **Nodes Involved:** 
  - On form submission
  - When Executed by Another Workflow
  - Sticky Note4 (documentation)
- **Node Details:**

  - **On form submission**
    - *Type:* Form Trigger
    - *Role:* Captures inputs interactively via a web form.
    - *Configuration:* 
      - Form titled "Extract comments from a specific post on X".
      - Fields: "X Post URL" (required), "Airtop Profile (connected to X)" (required), and "Max number of comments" (optional number).
      - Webhook ID assigned for external triggering.
    - *Expressions/Variables:* Inputs are accessed as JSON keys.
    - *Inputs:* None (trigger node).
    - *Outputs:* Passes form data downstream.
    - *Failure Modes:* Invalid or missing required inputs; webhook connection issues.

  - **When Executed by Another Workflow**
    - *Type:* Execute Workflow Trigger
    - *Role:* Allows this workflow to be executed programmatically from other workflows with input parameters.
    - *Configuration:* Defines expected workflow inputs: x_post_url, airtop_profile, max_number_of_comments.
    - *Inputs:* None (trigger node).
    - *Outputs:* Emits received parameters downstream.
    - *Failure Modes:* Missing inputs or incompatible data types.

  - **Sticky Note4**
    - *Type:* Sticky Note
    - *Role:* Provides documentation for input parameters and usage modes.
    - *Content:* "Run this workflow using a form or from another workflow"
    - *Inputs/Outputs:* None

#### 2.2 Parameter Normalization

- **Overview:** This block unifies input parameter naming conventions regardless of the source to ensure consistent downstream processing.
- **Nodes Involved:** 
  - Unify params
- **Node Details:**

  - **Unify params**
    - *Type:* Set Node
    - *Role:* Assigns normalized parameter names.
    - *Configuration:* 
      - Extracts "X Post URL" or "x_post_url" into `x_post_url`.
      - Extracts "Airtop Profile (connected to X)" or `airtop_profile` into `airtop_profile`.
      - Extracts "Max number of comments" or `max_number_of_comments` into `max_number_of_comments`.
    - *Expressions:* Uses expressions to fallback between possible input field names.
    - *Inputs:* From either On form submission or When Executed by Another Workflow.
    - *Outputs:* Normalized JSON with unified keys.
    - *Failure Modes:* Missing or malformed inputs may lead to empty or undefined normalized variables.

#### 2.3 Comment Extraction via Airtop

- **Overview:** This block leverages the Airtop node to create a browser session that visits the specified X post and extracts structured comment data based on a prompt.
- **Nodes Involved:** 
  - Extract X Post Comments
  - Edit Fields
- **Node Details:**

  - **Extract X Post Comments**
    - *Type:* Airtop Browser Automation Node
    - *Role:* Opens a new browser session, loads the X post URL, and runs a prompt-based extraction of comments.
    - *Configuration:*
      - URL set dynamically from `x_post_url`.
      - Prompt instructs extraction of up to `max_number_of_comments` comments, each with author name, profile URL, and comment text.
      - Resource: extraction.
      - ProfileName: set dynamically from `airtop_profile`.
      - Session Mode: new (starts fresh browser session).
      - Output Schema: JSON schema defining the expected structure of extracted comments.
      - Pagination Mode: infinite-scroll (handles loading more comments dynamically).
    - *Credentials:* Airtop API key linked to an official organization.
    - *Inputs:* Normalized parameters.
    - *Outputs:* JSON object with extracted comments array.
    - *Failure Modes:* 
      - Authentication errors if Airtop API key or profile is invalid.
      - Network or timeout errors during page load or extraction.
      - Schema validation errors if extracted data does not conform to expected structure.
      - Pagination issues if infinite scroll fails to load all comments.
    - *Version:* 1.0

  - **Edit Fields**
    - *Type:* Set Node
    - *Role:* Refines the output by explicitly assigning the extracted `modelResponse` data to a simplified `data.modelResponse` field.
    - *Configuration:* Copies `$json.data.modelResponse` to `data.modelResponse`.
    - *Inputs:* Output from Extract X Post Comments.
    - *Outputs:* Reformatted JSON object.
    - *Failure Modes:* Missing or malformed input JSON could result in empty or undefined output.

#### 2.4 Documentation and Metadata

- **Overview:** Provides comprehensive usage instructions, use cases, and setup requirements as a sticky note within the workflow.
- **Nodes Involved:** 
  - Sticky Note
- **Node Details:**

  - **Sticky Note**
    - *Type:* Sticky Note
    - *Role:* Full documentation embedded inside the workflow for end-users.
    - *Content:* Includes README-like details covering use case, input parameters, workflow operation, setup requirements, and next steps.
    - *Inputs/Outputs:* None

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                       | Input Node(s)                         | Output Node(s)           | Sticky Note                                                                                                                           |
|----------------------------|-----------------------------|------------------------------------|-------------------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                | Collects user input via form       | None                                | Unify params             | Run this workflow using a form or from another workflow                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger    | Allows execution from other workflows | None                                | Unify params             | Run this workflow using a form or from another workflow                                                                              |
| Unify params              | Set                         | Normalizes input parameters        | On form submission, When Executed by Another Workflow | Extract X Post Comments |                                                                                                                                     |
| Extract X Post Comments    | Airtop Browser Automation   | Extracts comments from X post      | Unify params                        | Edit Fields              |                                                                                                                                     |
| Edit Fields               | Set                         | Formats extracted data             | Extract X Post Comments             | None                     |                                                                                                                                     |
| Sticky Note4              | Sticky Note                 | Documents input usage              | None                                | None                     | Run this workflow using a form or from another workflow                                                                              |
| Sticky Note               | Sticky Note                 | Full workflow documentation        | None                                | None                     | README: Extracting Comments from an X Post. Includes use case, setup, operation, and next steps. See https://portal.airtop.ai/browser-profiles |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "On form submission":**  
   - Set form title to "Extract comments from a specific post on X".  
   - Add required fields:  
     - "X Post URL" (single-line text, required).  
     - "Airtop Profile (connected to X)" (single-line text, required).  
     - "Max number of comments" (number, optional).  
   - Note the webhook ID generated for triggering.

2. **Create an Execute Workflow Trigger node named "When Executed by Another Workflow":**  
   - Define workflow inputs:  
     - `x_post_url` (string).  
     - `airtop_profile` (string).  
     - `max_number_of_comments` (number).  

3. **Create a Set node named "Unify params":**  
   - Add assignments to normalize inputs:  
     - `x_post_url`: `{{$json["X Post URL"] || $json.x_post_url}}`  
     - `airtop_profile`: `{{$json["Airtop Profile (connected to X)"] || $json.airtop_profile}}`  
     - `max_number_of_comments`: `{{$json["Max number of comments"] || $json.max_number_of_comments}}`  
   - Connect both "On form submission" and "When Executed by Another Workflow" nodes to this node.

4. **Create an Airtop node named "Extract X Post Comments":**  
   - Set parameters:  
     - URL: `={{ $json.x_post_url }}`  
     - Prompt: `"This is an x post. Extract up to {{ $json.max_number_of_comments }} comments to the post. For each comment extract the name of the author, the x profile URL of the author, and the text of the comment."`  
     - Resource: `extraction`  
     - Profile Name: `={{ $json.airtop_profile }}`  
     - Session Mode: `new`  
     - Additional Fields:  
       - Output Schema: Use the provided JSON schema specifying an object with a `comments` array, where each comment includes `author_name`, `author_profile_url`, and `comment_text`.  
       - Pagination Mode: `infinite-scroll`  
   - Assign the Airtop API credentials (use an Airtop API key connected to your organization).

5. **Create a Set node named "Edit Fields":**  
   - Assign `data.modelResponse` to `{{$json.data.modelResponse}}` from the incoming JSON, preserving the extraction result in a simplified field.

6. **Connect nodes in order:**  
   - "On form submission" → "Unify params"  
   - "When Executed by Another Workflow" → "Unify params"  
   - "Unify params" → "Extract X Post Comments"  
   - "Extract X Post Comments" → "Edit Fields"

7. **Add Sticky Notes for documentation:**  
   - Create one sticky note titled "Sticky Note4" near input nodes to indicate input usage.  
   - Create a larger sticky note summarizing the workflow README with use case, setup, and instructions near the main workflow area.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This automation requires an Airtop API key available for free at https://portal.airtop.ai/api-keys and an Airtop Profile connected to X (one-time login required).                                                                        | Setup Requirements                                                                                |
| The output schema defines the structure of extracted comments strictly to ensure consistent downstream processing.                                                                                                                      | Extraction Node Configuration                                                                    |
| Pair this workflow with X Monitoring automation to automatically detect new posts and extract comments for sentiment or lead analysis. See https://www.airtop.ai/blog/automating-x-monitoring-with-airtop-and-make for reference.         | Suggested Next Steps                                                                             |
| Export the structured comment data into CRM or BI tools for enhanced audience insights and reporting.                                                                                                                                     | Suggested Next Steps                                                                             |
| Workflow documentation is embedded as sticky notes within the workflow for user reference.                                                                                                                                                 | Sticky Notes                                                                                     |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.