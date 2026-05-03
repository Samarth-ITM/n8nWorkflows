Extract Linkedin Company data with Airtop

https://n8nworkflows.xyz/workflows/extract-linkedin-company-data-with-airtop-4261


# Extract Linkedin Company data with Airtop

### 1. Workflow Overview

This workflow automates the extraction of detailed company information from a LinkedIn company page using Airtop’s AI-powered data extraction capabilities. It targets users who need structured, reliable, and actionable company insights such as investors, sales teams, and market researchers.

**Target Use Cases:**
- Extract comprehensive LinkedIn company profiles including identity, scale, classification, and funding data.
- Automate data collection for CRM enrichment, lead prioritization, or market analysis.
- Enable integration with other workflows or manual form submissions.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts company LinkedIn URL and Airtop profile via a web form or a trigger from another workflow.
- **1.2 Parameter Normalization:** Unifies input parameters into consistent variable names regardless of input source.
- **1.3 Airtop Data Extraction:** Uses Airtop node to perform the LinkedIn company data query with a detailed prompt, returning JSON-structured information.
- **1.4 Output Mapping:** Extracts and formats the relevant response data from Airtop’s output for downstream use.
- **1.5 Documentation and User Guidance:** Sticky notes provide contextual instructions, use cases, and setup requirements.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

**Overview:**  
This block triggers the workflow either via a form submission or by execution from another workflow, capturing the essential inputs: the LinkedIn company URL and the Airtop profile.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Listens for user input via a web form titled "Company information."  
  - Configuration:  
    - Fields:  
      - "Company's LinkedIn URL" (required, placeholder provided)  
      - "Airtop Profile (connected to Linkedin)" (required)  
    - Description: Explains the automation purpose and benefits.  
  - Inputs: External user submits form.  
  - Outputs: JSON object containing form fields.  
  - Edge Cases: Missing required fields triggers form validation errors; malformed URLs could cause downstream failures.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be invoked by another workflow, passing parameters programmatically.  
  - Configuration: Accepts two inputs: "company_linkedin" and "airtop_profile" as variables.  
  - Inputs: Workflow trigger with parameters.  
  - Outputs: JSON object with input parameters.  
  - Edge Cases: Missing or invalid parameters could cause extraction failure.

---

#### Block 1.2: Parameter Normalization

**Overview:**  
This block standardizes input parameters from either input method to a consistent internal format, ensuring downstream nodes receive uniform variable names.

**Nodes Involved:**  
- Unify Params

**Node Details:**

- **Unify Params**  
  - Type: Set Node  
  - Role: Assigns variables "company_linkedin" and "airtop_profile" from whichever input source is available.  
  - Configuration:  
    - Sets "company_linkedin" to `$json.company_linkedin` or falls back to `$json["Company's LinkedIn URL"]` (form field).  
    - Sets "airtop_profile" to `$json.airtop_profile` or `$json["Airtop Profile (connected to Linkedin)"]`.  
  - Inputs: JSON from either trigger node.  
  - Outputs: Clean JSON with consistent keys.  
  - Edge Cases: Both fields must be present; missing values cause errors in subsequent query node.

---

#### Block 1.3: Airtop Data Extraction

**Overview:**  
This block queries Airtop’s AI extraction engine with a detailed prompt to analyze and extract comprehensive company data from the provided LinkedIn URL using the specified Airtop profile.

**Nodes Involved:**  
- Extract Company's information

**Node Details:**

- **Extract Company's information**  
  - Type: Airtop Node (custom integration)  
  - Role: Executes an AI-powered query to extract structured company data from LinkedIn.  
  - Configuration:  
    - URL parameter set to `company_linkedin` variable.  
    - Prompt includes detailed instructions requesting data points: company profile, scale, classification, and funding info in JSON format.  
    - Uses "extraction" resource with "query" operation.  
    - Profile name supplied from `airtop_profile`.  
    - Session Mode: new (fresh session per execution).  
    - Output schema strictly defined to enforce JSON structure compliance.  
  - Inputs: Unified parameters with LinkedIn URL and profile.  
  - Outputs: Raw response containing JSON data under `data.modelResponse`.  
  - Credentials: Airtop API Key and linked Airtop Profile required.  
  - Edge Cases:  
    - Invalid URL or inaccessible LinkedIn page leads to empty or error responses.  
    - Airtop API quota or authentication failure possible.  
    - Prompt parsing errors or unexpected schema violations might cause failures.  
  - Version: v1

---

#### Block 1.4: Output Mapping

**Overview:**  
Formats the raw Airtop output to isolate the relevant JSON data for further processing or external consumption.

**Nodes Involved:**  
- Map output

**Node Details:**

- **Map output**  
  - Type: Set Node  
  - Role: Extracts and outputs only the `modelResponse` JSON object from the Airtop API response.  
  - Configuration:  
    - Uses expression: `{{$json.data.modelResponse}}` to map output.  
    - Outputs raw mode with no additional processing.  
  - Inputs: Airtop raw response.  
  - Outputs: Clean JSON structured company data.  
  - Edge Cases: Missing or malformed `modelResponse` field causes empty output or errors.

---

#### Block 1.5: Documentation and User Guidance

**Overview:**  
Sticky Notes provide useful context, instructions, and references for users interacting with or maintaining the workflow.

**Nodes Involved:**  
- Sticky Note4  
- Sticky Note  
- Sticky Note7

**Node Details:**

- **Sticky Note4**  
  - Content: "Input Parameters: Run this workflow using a form or from another workflow"  
  - Purpose: Clarifies input methods.

- **Sticky Note**  
  - Content: "Extract company information from LinkedIn"  
  - Purpose: Summarizes workflow function.

- **Sticky Note7**  
  - Content: Extensive README content including use case, output details, setup requirements, and next steps.  
  - Contains links:  
    - Airtop Profiles: https://portal.airtop.ai/browser-profiles  
    - Airtop API Key: https://portal.airtop.ai/api-keys  
  - Purpose: Full documentation for users and developers to understand the workflow scope and setup.

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                      |
|-----------------------------|----------------------------|----------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger               | Receives user input via form            | -                           | Unify Params               | Input Parameters: Run this workflow using a form or from another workflow                                        |
| When Executed by Another Workflow | Execute Workflow Trigger  | Receives input from other workflows    | -                           | Unify Params               | Input Parameters: Run this workflow using a form or from another workflow                                        |
| Unify Params                | Set                        | Normalizes input parameters             | On form submission, When Executed by Another Workflow | Extract Company's information |                                                                                                                 |
| Extract Company's information | Airtop                    | Extracts detailed company data from LinkedIn | Unify Params                | Map output                 | Extract company information from LinkedIn                                                                       |
| Map output                 | Set                        | Maps and formats Airtop output JSON    | Extract Company's information | -                          |                                                                                                                 |
| Sticky Note4               | Sticky Note                | Documentation: input parameters         | -                           | -                          | Input Parameters: Run this workflow using a form or from another workflow                                        |
| Sticky Note                | Sticky Note                | Documentation: workflow purpose         | -                           | -                          | Extract company information from LinkedIn                                                                       |
| Sticky Note7               | Sticky Note                | Documentation: detailed README          | -                           | -                          | README with use case, setup instructions, and links to Airtop profiles and API keys                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Form Trigger" node:**
   - Name it "On form submission".
   - Configure the form title as "Company information".
   - Add two form fields:
     - "Company's LinkedIn URL" (required, placeholder: https://www.linkedin.com/company/airtop-ai/).
     - "Airtop Profile (connected to Linkedin)" (required).
   - Add a form description explaining the automation goal.

3. **Add an "Execute Workflow Trigger" node:**
   - Name it "When Executed by Another Workflow".
   - Configure it to accept two input parameters: "company_linkedin" and "airtop_profile".

4. **Add a "Set" node named "Unify Params":**
   - Configure two assignments:
     - `company_linkedin`: Expression `{{$json.company_linkedin || $json["Company's LinkedIn URL"]}}`
     - `airtop_profile`: Expression `{{$json.airtop_profile || $json["Airtop Profile (connected to Linkedin)"]}}`
   - Connect outputs from both "On form submission" and "When Executed by Another Workflow" nodes into this node.

5. **Add an "Airtop" node named "Extract Company's information":**
   - Set resource to "extraction", operation to "query".
   - Set the "url" parameter to `{{$json.company_linkedin}}`.
   - Enter the detailed prompt requesting comprehensive LinkedIn company data extraction in structured JSON format (copy the prompt from original workflow).
   - Set "profileName" to `{{$json.airtop_profile}}`.
   - Set "sessionMode" to "new".
   - Paste the JSON schema for output validation as in the original.
   - Attach Airtop API credentials (API Key and Profile linked to LinkedIn).

6. **Connect "Unify Params" node output to "Extract Company's information" node.**

7. **Add a "Set" node named "Map output":**
   - Set mode to "raw".
   - Set JSON output to `{{$json.data.modelResponse}}` to extract the actual JSON response.
   - Connect "Extract Company's information" output to this node.

8. **Optionally, add Sticky Note nodes:**
   - Add notes explaining input methods, workflow purpose, and detailed README including setup links:
     - Airtop Profiles: https://portal.airtop.ai/browser-profiles
     - Airtop API Key: https://portal.airtop.ai/api-keys

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This automation requires an Airtop API Key and a LinkedIn-authenticated Airtop Browser Profile to function correctly.                                                                                                                                                                                                                                                                                                                                                                                     | https://portal.airtop.ai/api-keys and https://portal.airtop.ai/browser-profiles |
| The workflow extracts structured JSON data including company identity, scale, classification, and funding profile from LinkedIn company pages using AI. Recommended for CRM enrichment, sales lead prioritization, and market research.                                                                                                                                                                                                                                                                | Workflow purpose and use case notes                          |
| The prompt used in Airtop node is detailed and expects a JSON response matching a strict schema to ensure data consistency and ease of integration.                                                                                                                                                                                                                                                                                                                                                       | Internal workflow configuration                              |
| For best results, ensure LinkedIn URLs are public and valid, and Airtop profiles are correctly connected and authenticated with LinkedIn.                                                                                                                                                                                                                                                                                                                                                                  | Operational best practice                                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, which adheres strictly to content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly available.