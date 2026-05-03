Verify Linkedin Company Page by Domain with Airtop

https://n8nworkflows.xyz/workflows/verify-linkedin-company-page-by-domain-with-airtop-4263


# Verify Linkedin Company Page by Domain with Airtop

### 1. Workflow Overview

This workflow, titled **"Verify Company LinkedIn Page by Domain"**, is designed to validate if a LinkedIn company page URL corresponds to a specified company domain. It targets use cases such as lead qualification, CRM data enrichment, and ensuring the accuracy of LinkedIn URLs before storing or acting upon them.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives company LinkedIn URL, company domain, and Airtop profile either via a web form submission or from another workflow trigger.
- **1.2 Parameter Unification**: Normalizes and maps input parameters to internal variable names for consistent handling.
- **1.3 Website Extraction via Airtop**: Uses the Airtop node with a LinkedIn-authenticated profile to extract the company website URL from the LinkedIn company page.
- **1.4 Verification Filter**: Compares the extracted website URL against the provided company domain to confirm a valid match.
- **1.5 Output Mapping**: Prepares and returns the confirmed LinkedIn URL if validation succeeds.
- **1.6 Documentation and Notes**: Provides detailed instructions, use cases, and setup notes via sticky notes embedded in the workflow for user reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures essential inputs required for the verification process. It supports two entry points: via a web form or as an execution trigger from another workflow.

- **Nodes Involved:**  
  - `On form submission` (Form Trigger)  
  - `When Executed by Another Workflow` (Execute Workflow Trigger)  
  - `Sticky Note1` (Documentation)

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Captures user input from a web form titled "Verify the company LinkedIn"  
    - Configuration: Requires three fields: "Company LinkedIn", "Company Domain", and "Airtop Profile (connected to Linkedin)". The form description explains the automation's purpose.  
    - Inputs: HTTP POST from user form  
    - Outputs: JSON object containing form field values  
    - Edge cases: Missing required fields, malformed URLs, or unauthorized form access could cause failures.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows this workflow to be triggered programmatically from another workflow with input parameters.  
    - Configuration: Expects three inputs named identically to the form fields.  
    - Inputs: Workflow execution with input data  
    - Outputs: JSON object with input parameters  
    - Edge cases: Incorrect or missing workflow inputs may cause improper downstream processing.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Documents that the workflow can be run via form or another workflow, aiding user understanding.

#### 2.2 Parameter Unification

- **Overview:**  
  This block standardizes and renames the input data fields into internal variables for uniform access downstream.

- **Nodes Involved:**  
  - `Unify Params` (Set Node)

- **Node Details:**  

  - **Unify Params**  
    - Type: Set  
    - Role: Assigns input fields to variables: `linkedin`, `domain`, and `airtop_profile` for internal use.  
    - Configuration: Maps `"Company LinkedIn"` → `linkedin`, `"Company Domain"` → `domain`, `"Airtop Profile (connected to Linkedin)"` → `airtop_profile`.  
    - Inputs: JSON with original input keys  
    - Outputs: JSON with unified keys  
    - Edge cases: If input fields are missing or empty, variables may be undefined, causing failures downstream.

#### 2.3 Website Extraction via Airtop

- **Overview:**  
  Uses Airtop's API with a LinkedIn-authenticated profile to extract the official company website URL from the provided LinkedIn company page.

- **Nodes Involved:**  
  - `Get company website from LinkedIn profile` (Airtop Node)  
  - `Sticky Note` (Documentation note on LinkedIn URL verification)

- **Node Details:**  

  - **Get company website from LinkedIn profile**  
    - Type: Airtop  
    - Role: Queries Airtop extraction API to scrape the website URL from the LinkedIn company page.  
    - Configuration:  
      - URL set dynamically from unified param `linkedin`  
      - Airtop prompt: "This is a Company's LinkedIn profile page, extract the URL for the website."  
      - Uses the `airtop_profile` input for LinkedIn authentication  
      - Session mode set to "new"  
      - Airtop API credentials configured (requires valid Airtop API key and profile)  
    - Inputs: JSON with `linkedin` and `airtop_profile`  
    - Outputs: JSON containing `data.modelResponse` with extracted URL string  
    - Edge cases: Airtop API failures, invalid or expired credentials, rate limiting, or unexpected LinkedIn page layouts may cause extraction errors.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Reminds user to ensure the LinkedIn URL corresponds to the company to avoid false positives.

#### 2.4 Verification Filter

- **Overview:**  
  Compares the extracted website URL with the provided company domain to confirm if the LinkedIn page belongs to the expected company.

- **Nodes Involved:**  
  - `Filter` (Filter Node)  
  - `Map response` (Set Node)

- **Node Details:**  

  - **Filter**  
    - Type: Filter  
    - Role: Checks if the extracted website URL (`data.modelResponse`) contains the expected company domain (`domain`).  
    - Configuration:  
      - Condition uses "string contains" operator on `data.modelResponse` with `domain`  
      - Case sensitive and strict type validation enabled  
      - Combines conditions with "OR" (only one condition present)  
    - Inputs: JSON with extracted website URL and unified parameters  
    - Outputs: Passes data forward if condition matches; otherwise, data is filtered out  
    - Edge cases: Case sensitivity might cause false negatives; partial domain matches or subdomains may cause false positives.

  - **Map response**  
    - Type: Set  
    - Role: Prepares the final output by setting the confirmed LinkedIn URL under `company_linkedin`.  
    - Configuration: Sets `company_linkedin` to the original `linkedin` input value.  
    - Inputs: Filter-passed JSON  
    - Outputs: JSON containing verified LinkedIn URL  
    - Edge cases: If the filter blocks data, this node never executes.

#### 2.5 Documentation and Notes

- **Overview:**  
  Provides detailed readme-style information embedded inside sticky notes for users to understand the workflow's purpose, setup, and usage scenarios.

- **Nodes Involved:**  
  - `Sticky Note7` (Large documentation note)

- **Node Details:**  

  - **Sticky Note7**  
    - Type: Sticky Note  
    - Role: Contains extensive README content explaining the use case, input/output parameters, setup requirements, and next steps.  
    - Content includes links to Airtop API keys and profile setup, and guidance on integrating this workflow into data pipelines or CRM updates.  
    - No inputs or outputs, purely informational.

---

### 3. Summary Table

| Node Name                          | Node Type                   | Functional Role                            | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                               |
|-----------------------------------|-----------------------------|-------------------------------------------|--------------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------------------|
| On form submission                | Form Trigger                | Receive user input via web form            | -                                    | Unify Params                         | ## Input Parameters Run this workflow using a form or from another workflow                              |
| When Executed by Another Workflow | Execute Workflow Trigger    | Receive inputs from other workflow         | -                                    | Unify Params                         | ## Input Parameters Run this workflow using a form or from another workflow                              |
| Unify Params                     | Set                         | Normalize and assign input parameters      | On form submission, When Executed by Another Workflow | Get company website from LinkedIn profile |                                                                                                          |
| Get company website from LinkedIn profile | Airtop                      | Extract company website URL from LinkedIn  | Unify Params                         | Filter                              | ## Verify LinkedIn URL Ensure that the provided link corresponds to the company.                         |
| Filter                          | Filter                      | Verify if extracted website contains domain | Get company website from LinkedIn profile | Map response                       |                                                                                                          |
| Map response                   | Set                         | Prepare output with verified LinkedIn URL  | Filter                              | -                                    |                                                                                                          |
| Sticky Note1                   | Sticky Note                 | Documentation on input methods              | -                                    | -                                    | ## Input Parameters Run this workflow using a form or from another workflow                              |
| Sticky Note                    | Sticky Note                 | Documentation on verification importance    | -                                    | -                                    | ## Verify LinkedIn URL Ensure that the provided link corresponds to the company.                         |
| Sticky Note7                  | Sticky Note                 | Comprehensive README and usage instructions | -                                    | -                                    | README\n\n# Automating LinkedIn Company URL Verification\n\n## Use Case\n\nThis automation verifies...   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configure:  
     - Form Title: "Verify the company LinkedIn"  
     - Form Description: "This automation verifies whether a LinkedIn URL belongs to a company, based on its domain"  
     - Fields:  
       - Company LinkedIn (Required)  
       - Company Domain (Required)  
       - Airtop Profile (connected to Linkedin) (Required)  
   - Position: (0, -40)

2. **Create an Execute Workflow Trigger Node**  
   - Name: `When Executed by Another Workflow`  
   - Type: Execute Workflow Trigger  
   - Configure Input Parameters:  
     - Company LinkedIn  
     - Company Domain  
     - Airtop Profile (connected to Linkedin)  
   - Position: (0, 160)

3. **Create a Set Node for Parameter Unification**  
   - Name: `Unify Params`  
   - Type: Set  
   - Configure Assignments:  
     - `linkedin` = `{{ $json["Company LinkedIn"] }}`  
     - `domain` = `{{ $json["Company Domain"] }}`  
     - `airtop_profile` = `{{ $json["Airtop Profile (connected to Linkedin)"] }}`  
   - Position: (220, 60)  
   - Connect outputs of both `On form submission` and `When Executed by Another Workflow` to this node.

4. **Create Airtop Node to Extract Website**  
   - Name: `Get company website from LinkedIn profile`  
   - Type: Airtop  
   - Configure:  
     - URL: `{{ $json.linkedin }}`  
     - Prompt: "This is a Company's LinkedIn profile page, extract the URL for the website."  
     - Resource: Extraction  
     - Operation: Query  
     - Profile Name: `{{ $json.airtop_profile }}`  
     - Session Mode: New  
     - Additional Fields: leave empty  
     - Auto Terminate Session: false  
   - Credentials: Set Airtop API credentials with valid API key and profile  
   - Position: (440, 60)  
   - Connect `Unify Params` output to this node.

5. **Create Filter Node to Validate Domain**  
   - Name: `Filter`  
   - Type: Filter  
   - Configure Condition:  
     - Operator: String contains  
     - Left Value: `{{ $json.data.modelResponse }}`  
     - Right Value: `{{ $('Unify Params').item.json.domain }}`  
     - Case Sensitive: true  
     - Type Validation: strict  
     - Combinator: OR (only one condition)  
   - Position: (660, 60)  
   - Connect `Get company website from LinkedIn profile` output to this node.

6. **Create Set Node to Map Verified Output**  
   - Name: `Map response`  
   - Type: Set  
   - Configure Assignments:  
     - `company_linkedin` = `{{ $('Unify Params').item.json.linkedin }}`  
   - Position: (880, 60)  
   - Connect `Filter` output (main output, if pass) to this node.

7. **Add Sticky Notes for Documentation**  
   - Create Sticky Note near Input nodes (positioned around -40, -140): Content:  
     ```
     ## Input Parameters
     Run this workflow using a form or from another workflow
     ```  
   - Create Sticky Note near Airtop node (positioned around 380, -140): Content:  
     ```
     ## Verify LinkedIn URL
     Ensure that the provided link corresponds to the company.
     ```  
   - Create Large Sticky Note for README near bottom left (positioned around -760, -500): Content includes detailed README as in the original, explaining use cases, setup, and next steps.

8. **Connect Nodes as Follows:**  
   - `On form submission` → `Unify Params`  
   - `When Executed by Another Workflow` → `Unify Params`  
   - `Unify Params` → `Get company website from LinkedIn profile`  
   - `Get company website from LinkedIn profile` → `Filter`  
   - `Filter` → `Map response` (if passes)

9. **Credential Setup:**  
   - Configure Airtop API credentials with a valid API key from [Airtop API Keys](https://portal.airtop.ai/api-keys)  
   - Ensure to create and connect an Airtop browser profile authenticated with LinkedIn at [Airtop Profiles](https://portal.airtop.ai/browser-profiles)  
   - Assign these credentials to the Airtop node.

10. **Defaults and Constraints:**  
    - All form fields and workflow inputs are required to avoid missing data errors.  
    - The filter uses case-sensitive matching; adjust if needed for your use case.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow is designed to verify LinkedIn company URLs by comparing extracted website URLs against expected company domains, critical for lead qualification and CRM data accuracy.                                                     | Workflow purpose                                              |
| Airtop profile used must be authenticated with LinkedIn to enable scraping of company website data from LinkedIn pages.                                                                                                            | [Airtop Browser Profiles](https://portal.airtop.ai/browser-profiles) |
| Requires Airtop API key for API access: [Airtop API Keys](https://portal.airtop.ai/api-keys)                                                                                                                                         | Credential setup                                             |
| README sticky note explains use cases, input/output parameters, and next steps including integration in data pipelines and CRM systems.                                                                                             | Embedded in workflow (Sticky Note7)                           |
| The workflow can be triggered by a web form or programmatically from another workflow, offering flexible integration options.                                                                                                       | Input block description                                      |

---

**Disclaimer**: The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.