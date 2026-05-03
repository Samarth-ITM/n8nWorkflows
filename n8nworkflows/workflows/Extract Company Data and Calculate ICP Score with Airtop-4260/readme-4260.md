Extract Company Data and Calculate ICP Score with Airtop

https://n8nworkflows.xyz/workflows/extract-company-data-and-calculate-icp-score-with-airtop-4260


# Extract Company Data and Calculate ICP Score with Airtop

### 1. Workflow Overview

This n8n workflow automates the process of extracting comprehensive company data and calculating an Ideal Customer Profile (ICP) score. It targets sales teams, data enrichment pipelines, and CRM systems aiming to qualify and enrich company records based on LinkedIn data and other business attributes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Accepts input parameters either from a form submission or another workflow execution trigger.
- **1.2 Parameter Unification**: Normalizes and unifies incoming parameters for consistent downstream processing.
- **1.3 LinkedIn URL Resolution**: Validates and resolves the company’s LinkedIn URL; if missing or invalid, triggers an external workflow to discover the URL.
- **1.4 Company Data Extraction**: Invokes an external workflow to extract detailed company information from LinkedIn using Airtop profiles.
- **1.5 ICP Score Calculation**: Invokes another external workflow to calculate the ICP score based on the enriched company data.
- **1.6 Results Aggregation**: Merges all collected data components into a single unified output.

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview**: This block receives input parameters either via a user form or from another workflow execution trigger. It acts as the entry point for the workflow.
- **Nodes Involved**:
  - On form submission
  - When Executed by Another Workflow
  - Sticky Note (Input Parameters description)
- **Node Details**:

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Captures user input from an embedded form titled "LinkedIn Company Profile Extractor".  
    - Configuration:  
      - Form fields: Company domain (required), Airtop Profile (required), Company LinkedIn (optional)  
      - Form description explains that the automation finds the LinkedIn URL from a company domain.  
    - Inputs: User form submission  
    - Outputs: Passes form data downstream  
    - Edge Cases: Missing required fields; invalid domain format; user cancels submission.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows external workflows to trigger this workflow with input parameters.  
    - Configuration: Accepts inputs for Company domain, Airtop Profile, Company LinkedIn.  
    - Inputs: Triggered externally by other workflows  
    - Outputs: Passes input parameters downstream  
    - Edge Cases: Missing inputs; incompatible data types.

  - **Sticky Note**  
    - Provides instructions that the workflow can be run through form submission or another workflow.

#### 2.2 Parameter Unification

- **Overview**: Normalizes incoming parameters from the form or external workflow trigger to a consistent JSON structure for internal processing.
- **Nodes Involved**:
  - Unify Params
- **Node Details**:

  - **Unify Params**  
    - Type: Set  
    - Role: Assigns normalized field names (`company_domain`, `airtop_profile`, `company_linkedin`) from incoming data fields.  
    - Configuration: Uses expressions to extract data from previous nodes and maps them to normalized keys.  
    - Inputs: Data from either entry points  
    - Outputs: Unified JSON with normalized keys  
    - Edge Cases: Missing or empty values; expression evaluation errors.

#### 2.3 LinkedIn URL Resolution

- **Overview**: Checks if a valid LinkedIn company URL exists. If yes, proceeds; if no, triggers an external workflow to find the LinkedIn URL.
- **Nodes Involved**:
  - LinkedIn link exists? (If node)
  - Is valid LinkedIn link? (Filter node)
  - Extract company LinkedIn url (Execute Workflow)
  - Sticky Note2 (LinkedIn URL resolution explanation)
- **Node Details**:

  - **LinkedIn link exists?**  
    - Type: If  
    - Role: Checks if the `company_linkedin` field is non-empty.  
    - Configuration: Condition `company_linkedin` is not empty.  
    - Inputs: Unified parameters node output  
    - Outputs:  
      - True: Passes to validity check  
      - False: Passes to LinkedIn URL extraction workflow  
    - Edge Cases: Field missing or empty string.

  - **Is valid LinkedIn link?**  
    - Type: Filter  
    - Role: Validates if the LinkedIn URL contains "linkedin.com/company".  
    - Configuration: String contains check on `company_linkedin`.  
    - Inputs: Output from LinkedIn link exists?  
    - Outputs:  
      - True: Valid LinkedIn URL, proceeds to extraction and scoring  
      - False: Invalid URL, triggers extraction workflow  
    - Edge Cases: URL malformed; partial matches.

  - **Extract company LinkedIn url**  
    - Type: Execute Workflow  
    - Role: Runs an external workflow named "AIRTOP — Company LinkedIn" to find the company LinkedIn URL if missing or invalid.  
    - Configuration: Passes `company_domain` and `airtop_profile` as inputs.  
    - Inputs: Triggered if LinkedIn URL is missing or invalid  
    - Outputs: Returns a resolved `company_linkedin` URL  
    - Edge Cases: External workflow failure; API errors; no URL found.

  - **Sticky Note2**  
    - Explains the logic of finding the company LinkedIn URL, including fallback search.

#### 2.4 Company Data Extraction

- **Overview**: Uses an external workflow to extract detailed company information from the resolved LinkedIn URL.
- **Nodes Involved**:
  - Extract Company Information (Execute Workflow)
  - Sticky Note1 (Enrich Company Data description)
- **Node Details**:

  - **Extract Company Information**  
    - Type: Execute Workflow  
    - Role: Runs "AIRTOP — Extract LinkedIn Company Information" workflow to gather company profile data.  
    - Configuration: Inputs are `airtop_profile` and resolved `company_linkedin`.  
    - Inputs: Valid LinkedIn URL and Airtop profile  
    - Outputs: Structured company information such as name, tagline, website, location, about, employee count, classification.  
    - Edge Cases: API or data extraction failures; invalid inputs.

  - **Sticky Note1**  
    - Highlights this block’s function to enrich company data and calculate ICP score.

#### 2.5 ICP Score Calculation

- **Overview**: Executes a separate workflow that applies an ICP scoring model to the enriched company data.
- **Nodes Involved**:
  - Calclate ICP (Execute Workflow)
- **Node Details**:

  - **Calclate ICP**  
    - Type: Execute Workflow  
    - Role: Runs "AIRTOP — Company ICP scoring" to compute the Ideal Customer Profile score.  
    - Configuration: Inputs are `airtop_profile` and `company_linkedin`.  
    - Inputs: Same as company information extraction node  
    - Outputs: ICP score and detailed justifications for scoring  
    - Edge Cases: Workflow failures; scoring engine errors.

#### 2.6 Results Aggregation

- **Overview**: Merges all outputs from company information extraction and ICP scoring into a single JSON object.
- **Nodes Involved**:
  - Merge
- **Node Details**:

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from Extract Company Information, Calclate ICP, and Extract company LinkedIn url into one final dataset.  
    - Configuration: Number of inputs set to 3; combines data by merging JSON objects.  
    - Inputs: Outputs from data extraction, ICP calculation, and LinkedIn URL extraction workflows  
    - Outputs: Unified JSON containing all enriched company data and ICP score  
    - Edge Cases: Conflicting or missing fields in merges; data format inconsistencies.

#### Additional Nodes

- **Sticky Note3 (README)**  
  - Provides detailed documentation and context for the entire workflow, including use case, input/output descriptions, processing steps, setup requirements, and next steps for integrations.

- **Sticky Note (Input Parameters)**  
  - Explains the workflow can be run via form or external trigger.

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                              | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                 |
|----------------------------|----------------------------|----------------------------------------------|---------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger               | Entry point for user input via form           | -                               | Unify Params                      | Input Parameters: Run this workflow using a form or from another workflow                   |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point for external workflow trigger    | -                               | Unify Params                      | Input Parameters: Run this workflow using a form or from another workflow                   |
| Unify Params               | Set                       | Normalize input parameters                     | On form submission, When Executed by Another Workflow | LinkedIn link exists?             |                                                                                            |
| LinkedIn link exists?       | If                        | Checks if LinkedIn URL is provided            | Unify Params                    | Is valid LinkedIn link?, Extract company LinkedIn url |                                                                                            |
| Is valid LinkedIn link?     | Filter                    | Validates LinkedIn URL format                  | LinkedIn link exists?           | Extract Company Information, Merge, Calclate ICP, Extract company LinkedIn url |                                                                                            |
| Extract company LinkedIn url | Execute Workflow          | Finds LinkedIn URL if missing or invalid      | LinkedIn link exists? (False)   | Is valid LinkedIn link?           | Find Company LinkedIn URL: If the link isn't found on the page, search for it externally   |
| Extract Company Information | Execute Workflow          | Extracts detailed company LinkedIn data       | Is valid LinkedIn link? (True)  | Merge                            | Enrich Company Data: Extract all information and calculate ICP score                        |
| Calclate ICP               | Execute Workflow          | Computes ICP score based on company data      | Is valid LinkedIn link? (True)  | Merge                            | Enrich Company Data: Extract all information and calculate ICP score                        |
| Merge                      | Merge                     | Aggregates all data into unified output       | Extract Company Information, Calclate ICP, Extract company LinkedIn url | -                               |                                                                                            |
| Sticky Note                | Sticky Note               | Provides input parameter instructions          | -                               | -                               | Input Parameters: Run this workflow using a form or from another workflow                   |
| Sticky Note1               | Sticky Note               | Describes company data enrichment block       | -                               | -                               | Enrich Company Data: Extract all information and calculate ICP score                        |
| Sticky Note2               | Sticky Note               | Describes LinkedIn URL resolution block       | -                               | -                               | Find Company LinkedIn URL: If the link isn't found on the page, search for it externally   |
| Sticky Note3               | Sticky Note               | Detailed README and documentation              | -                               | -                               | README: Full project description, use case, setup, and next steps                          |

### 4. Reproducing the Workflow from Scratch

1. **Create 'On form submission' node**  
   - Type: Form Trigger  
   - Configure form titled "LinkedIn Company Profile Extractor" with fields:  
     - Company domain (required, placeholder "company.com")  
     - Airtop Profile (connected to LinkedIn, required)  
     - Company LinkedIn (optional)  
   - Add form description with HTML content explaining the purpose.

2. **Create 'When Executed by Another Workflow' node**  
   - Type: Execute Workflow Trigger  
   - Configure to accept inputs: Company domain, Airtop Profile (connected to LinkedIn), Company LinkedIn

3. **Create 'Unify Params' node**  
   - Type: Set  
   - Map incoming fields to normalized keys:  
     - `company_domain` ← "Company domain"  
     - `airtop_profile` ← "Airtop Profile (connected to Linkedin)"  
     - `company_linkedin` ← "Company LinkedIn"  
   - Use expressions to extract these from the previous nodes.

4. **Create 'LinkedIn link exists?' node**  
   - Type: If  
   - Condition: Check if `company_linkedin` is not empty.

5. **Create 'Is valid LinkedIn link?' node**  
   - Type: Filter  
   - Condition: `company_linkedin` contains substring "linkedin.com/company"

6. **Create 'Extract company LinkedIn url' node**  
   - Type: Execute Workflow  
   - Reference external workflow "AIRTOP — Company LinkedIn" (workflow ID: JuW1Q92YSk4ErvvC)  
   - Input parameters:  
     - `Company domain`: from `company_domain`  
     - `Airtop Profile (connected to Linkedin)`: from `airtop_profile`

7. **Create 'Extract Company Information' node**  
   - Type: Execute Workflow  
   - Reference external workflow "AIRTOP — Extract LinkedIn Company Information" (workflow ID: yWtpgGwGgspx1A1W)  
   - Input parameters:  
     - `airtop_profile`: from `airtop_profile`  
     - `company_linkedin`: from resolved LinkedIn URL

8. **Create 'Calclate ICP' node**  
   - Type: Execute Workflow  
   - Reference external workflow "AIRTOP — Company ICP scoring" (workflow ID: CYDemwO42LTGiiPR)  
   - Input parameters same as in step 7.

9. **Create 'Merge' node**  
   - Type: Merge  
   - Configure to accept 3 inputs: outputs from  
     - Extract Company Information node  
     - Calclate ICP node  
     - Extract company LinkedIn url node  
   - Set merge mode to combine JSON objects.

10. **Connect nodes in order:**  
    - From both "On form submission" and "When Executed by Another Workflow" → "Unify Params"  
    - "Unify Params" → "LinkedIn link exists?"  
    - "LinkedIn link exists?" True output → "Is valid LinkedIn link?"  
    - "LinkedIn link exists?" False output → "Extract company LinkedIn url"  
    - "Extract company LinkedIn url" → "Is valid LinkedIn link?"  
    - "Is valid LinkedIn link?" True output → "Extract Company Information" → "Merge"  
    - "Is valid LinkedIn link?" True output → "Calclate ICP" → "Merge"  
    - "Is valid LinkedIn link?" False output → "Extract company LinkedIn url" (loop covered above)  

11. **Add Sticky Notes for documentation:**  
    - Input parameters explanation near entry nodes  
    - LinkedIn URL resolution explanation near LinkedIn URL nodes  
    - Enrich company data and ICP scoring explanation near extraction and scoring nodes  
    - README note with full project description, use case, and setup instructions

12. **Credential setup:**  
    - Airtop Profile credentials must be configured in n8n credentials manager and linked in the respective external workflows.  
    - Ensure access and API keys for Airtop are valid and tested.  

13. **Test workflow:**  
    - Run using the form input or trigger from another workflow.  
    - Verify resolution of LinkedIn URL, data extraction, ICP score calculation, and merged output.

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automates company LinkedIn profile detection, data extraction, and ICP scoring for sales and CRM enrichment.                    | Project Description (Sticky Note3)                                                              |
| Requires Airtop API key and authenticated Airtop profile with LinkedIn access.                                                           | Setup Requirements in README (Sticky Note3)                                                     |
| Airtop Browser Profiles: https://portal.airtop.ai/browser-profiles                                                                         | Mentioned in workflow input descriptions                                                       |
| Airtop API Keys: https://portal.airtop.ai/api-keys                                                                                        | Mentioned in workflow setup requirements                                                       |
| Suggestion to integrate with person enrichment workflows and CRM syncing for full automation pipeline.                                   | README next steps (Sticky Note3)                                                                |
| ICP scoring logic is customizable to fit organizational models.                                                                           | README next steps (Sticky Note3)                                                                |

---

This documentation provides a detailed, structured understanding of the "Extract company data and calculate ICP" workflow, enabling efficient reproduction, modification, and error anticipation.