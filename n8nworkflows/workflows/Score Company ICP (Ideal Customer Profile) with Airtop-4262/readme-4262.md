Score Company ICP (Ideal Customer Profile) with Airtop

https://n8nworkflows.xyz/workflows/score-company-icp--ideal-customer-profile--with-airtop-4262


# Score Company ICP (Ideal Customer Profile) with Airtop

### 1. Workflow Overview

This workflow automates scoring a company’s Ideal Customer Profile (ICP) using its LinkedIn profile data, accessed and processed through Airtop. It’s designed primarily for B2B lead qualification, enabling sales or marketing teams to prioritize companies based on a composite score derived from specific criteria.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts input parameters via a form submission or invocation from another workflow.
- **1.2 Parameter Normalization:** Harmonizes input parameter names for consistent downstream use.
- **1.3 ICP Score Calculation:** Uses the Airtop node to analyze the LinkedIn profile and compute the ICP score based on predefined criteria.
- **1.4 Data Parsing and Flattening:** Parses the raw response into JSON and restructures it to output key numeric scores for integration or storage.
- **1.5 (Informational) Sticky Notes:** Documentation nodes providing embedded instructions, scoring rubric, and usage notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives inputs either from a user form submission or from another workflow trigger, ensuring flexible integration points.

- **Nodes Involved:**  
  - On form submission  
  - When Executed by Another Workflow

- **Node Details:**

  - **On form submission**  
    - *Type & Role:* Form Trigger node; listens for external form submissions to start the workflow.  
    - *Configuration:*  
      - Form titled "Company ICP scoring"  
      - Two fields: "Company LinkedIn URL" and "Airtop Profile (connected to Linkedin)"  
      - Description explains the automation purpose.  
    - *Input/Output:* Outputs form data as JSON when a submission occurs.  
    - *Edge Cases:* Missing or malformed URLs; empty Airtop profile input; webhook connectivity issues.

  - **When Executed by Another Workflow**  
    - *Type & Role:* Execute Workflow Trigger node; allows the workflow to be called programmatically with inputs.  
    - *Configuration:* Accepts two parameters: `company_linkedin` and `airtop_profile`.  
    - *Input/Output:* Triggers execution with provided inputs.  
    - *Edge Cases:* Missing input parameters; invalid data types.

---

#### 2.2 Parameter Normalization

- **Overview:**  
  Aligns input field names from either form submission or workflow execution into a consistent internal format with keys `company_linkedin` and `airtop_profile`.

- **Nodes Involved:**  
  - Unify params

- **Node Details:**

  - **Unify params**  
    - *Type & Role:* Set node; assigns standardized variable names.  
    - *Configuration:*  
      - Reads from JSON fields `Company LinkedIn URL` or `company_linkedin` and assigns to `company_linkedin`  
      - Reads from `Airtop Profile (connected to Linkedin)` or `airtop_profile` and assigns to `airtop_profile`  
    - *Input/Output:* Takes raw input JSON and outputs JSON with consistent keys.  
    - *Edge Cases:* Missing or empty input fields; expression errors if fields are undefined.

---

#### 2.3 ICP Score Calculation

- **Overview:**  
  Core logic block that invokes Airtop’s API to analyze the LinkedIn profile data and compute ICP scores using a detailed scoring rubric.

- **Nodes Involved:**  
  - Calculate ICP

- **Node Details:**

  - **Calculate ICP**  
    - *Type & Role:* Airtop node; acts as an AI-powered data extraction and scoring engine.  
    - *Configuration:*  
      - URL input: company LinkedIn profile URL (from `company_linkedin`)  
      - Prompt: Detailed instructions specifying data fields (About, Employee count, Industry, Headquarters, Services, Keywords) and scoring criteria to analyze.  
      - Profile: Airtop profile authenticated to LinkedIn (from `airtop_profile`)  
      - Session mode: New session  
      - Output schema: JSON schema defining expected numeric scores and justification strings for each scoring category (AI Focus, Technical Level, Employee Count, Agency Status, Geography) plus total score.  
    - *Input/Output:* Receives normalized inputs; outputs JSON data structured per schema.  
    - *Version Requirements:* Airtop credential must be configured and valid; n8n version supporting Airtop node v1.  
    - *Edge Cases:* API authentication failures; invalid LinkedIn URLs; insufficient profile data; prompt parsing errors; timeout or rate limits from Airtop API.

---

#### 2.4 Data Parsing and Flattening

- **Overview:**  
  Transforms the raw Airtop API response into usable JSON and flattens key scoring metrics into top-level variables for easy downstream use.

- **Nodes Involved:**  
  - Parse to JSON  
  - Flat json

- **Node Details:**

  - **Parse to JSON**  
    - *Type & Role:* Set node with raw mode; extracts the `modelResponse` string from Airtop response and parses it into JSON.  
    - *Configuration:*  
      - Reads JSON from `$json.data.modelResponse`  
      - Output is JSON only without other fields.  
    - *Edge Cases:* Malformed or empty `modelResponse`; parsing errors.

  - **Flat json**  
    - *Type & Role:* Set node; extracts individual scores from parsed JSON and assigns to numeric variables.  
    - *Configuration:* Assigns:  
      - `icp_company_score` = total_score  
      - `ai_focus`, `employee_count`, `technical_level`, `agency_status`, `geography` = corresponding category scores  
    - *Edge Cases:* Missing fields in parsed JSON; type coercion errors.

---

#### 2.5 Sticky Notes (Documentation)

- **Overview:**  
  Informational nodes providing embedded documentation on workflow usage, input parameters, and scoring criteria.

- **Nodes Involved:**  
  - Sticky Note4  
  - Sticky Note  
  - Sticky Note7

- **Node Details:**

  - **Sticky Note4**  
    - Content: "Input Parameters: Run this workflow using a form or from another workflow"  
    - Positioned near input nodes for clarity.

  - **Sticky Note**  
    - Content: "Calculate ICP" heading placed near the calculation node.

  - **Sticky Note7**  
    - Content: Extensive README-style note explaining:  
      - Use case and inputs  
      - Scoring rubric with point breakdowns  
      - Workflow operation overview  
      - Setup requirements including Airtop API key and profile  
      - Suggestions for next steps and integration possibilities  
      - Links to Airtop browser profiles and API keys.  
    - Positioned to cover the entire workflow area for context.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                             |
|---------------------------|--------------------------------|---------------------------------------|-----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger                   | Receive user input via form            |                             | Unify params               | Input Parameters: Run this workflow using a form or from another workflow                              |
| When Executed by Another Workflow | Execute Workflow Trigger        | Receive input when called from another workflow |                             | Unify params               | Input Parameters: Run this workflow using a form or from another workflow                              |
| Unify params              | Set                            | Normalize input parameter names        | On form submission, When Executed by Another Workflow | Calculate ICP             |                                                                                                       |
| Calculate ICP             | Airtop                         | Analyze LinkedIn profile and calculate ICP score | Unify params                 | Parse to JSON              | Calculate ICP                                                                                        |
| Parse to JSON             | Set (raw mode)                 | Parse raw Airtop response JSON string  | Calculate ICP                | Flat json                  |                                                                                                       |
| Flat json                 | Set                            | Flatten key ICP scores into variables  | Parse to JSON                |                            |                                                                                                       |
| Sticky Note4              | Sticky Note                    | Documentation: Input Parameters        |                             |                            | Input Parameters: Run this workflow using a form or from another workflow                              |
| Sticky Note               | Sticky Note                    | Documentation: Calculate ICP heading   |                             |                            | Calculate ICP                                                                                        |
| Sticky Note7              | Sticky Note                    | Extensive README and scoring rubric    |                             |                            | README: Automating Company ICP Scoring via LinkedIn with detailed use case, setup, and next steps      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**  
   - Name: `On form submission`  
   - Form Title: `Company ICP scoring`  
   - Form Description: "This automation takes company's Linkedin Profile URL and Airtop Profile (authenticated for Linkedin) and returns the company's ICP score"  
   - Add two form fields:  
     - "Company LinkedIn URL" (placeholder: https://www.linkedin.com/company/airtop-ai/)  
     - "Airtop Profile (connected to Linkedin)"  
   - Save and note the webhook URL generated.

3. **Add an Execute Workflow Trigger node:**  
   - Name: `When Executed by Another Workflow`  
   - Configure with two input parameters: `company_linkedin`, `airtop_profile`.

4. **Add a Set node:**  
   - Name: `Unify params`  
   - Create two fields by assignment:  
     - `company_linkedin` with expression: `{{$json["Company LinkedIn URL"] || $json.company_linkedin}}`  
     - `airtop_profile` with expression: `{{$json["Airtop Profile (connected to Linkedin)"] || $json.airtop_profile}}`

5. **Connect both `On form submission` and `When Executed by Another Workflow` nodes to `Unify params`.**

6. **Add an Airtop node:**  
   - Name: `Calculate ICP`  
   - Credentials: Select your Airtop API key credential (e.g., "Airtop Official Org")  
   - Parameters:  
     - URL: `={{$json.company_linkedin}}`  
     - Prompt: Copy the detailed prompt instructing the AI to analyze the LinkedIn profile and score by categories (AI Focus, Technical Level, Employee Count, Agency Status, Geography) with justification and total score calculation.  
     - Resource: `extraction`  
     - Operation: `query`  
     - Profile Name: `={{$json.airtop_profile}}`  
     - Session Mode: `new`  
     - Additional Fields: Set output schema to the detailed JSON schema defining integer scores and justifications as per the prompt.

7. **Connect `Unify params` node’s output to `Calculate ICP`.**

8. **Add a Set node:**  
   - Name: `Parse to JSON`  
   - Mode: Raw  
   - JSON Output: `={{ $json.data.modelResponse }}`  
   - Include Other Fields: false

9. **Connect `Calculate ICP` to `Parse to JSON`.**

10. **Add another Set node:**  
    - Name: `Flat json`  
    - Create assignments for:  
      - `icp_company_score` = `={{ $json.total_score }}`  
      - `ai_focus` = `={{ $json.AI_Focus }}`  
      - `employee_count` = `={{ $json.Employee_Count }}`  
      - `technical_level` = `={{ $json.Technical_Level }}`  
      - `agency_status` = `={{ $json.Agency_Status }}`  
      - `geography` = `={{ $json.Geography }}`

11. **Connect `Parse to JSON` to `Flat json`.**

12. **Optionally, add Sticky Note nodes for in-editor documentation:**  
    - Near input nodes: "Run this workflow using a form or from another workflow"  
    - Near calculation node: "Calculate ICP"  
    - Large note covering the workflow area with the README content explaining use case, scoring rubric, and setup links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This automation scores companies based on their LinkedIn profile using custom Ideal Customer Profile (ICP) criteria. It is ideal for qualifying B2B leads and prioritizing outreach based on fit. Input parameters include Company LinkedIn URL and Airtop Profile authenticated to LinkedIn. The scoring rubric covers AI Focus, Technical Level, Employee Count, Agency Status, and Geography, with points assigned per category and a total composite score returned alongside justifications. Setup requires Airtop API Key and a LinkedIn-authenticated Airtop profile. | https://portal.airtop.ai/api-keys and https://portal.airtop.ai/browser-profiles                             |
| The workflow supports two execution paths: via a user-submitted form or invocation from another workflow, providing flexibility for integration in larger automation ecosystems.                                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                |
| Adjust scoring weights or categories in the Airtop node prompt and output schema to customize ICP criteria to your business needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |                                                                                                |
| Consider combining this workflow with CRM update nodes (e.g., HubSpot, Salesforce) to push scores for lead prioritization.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |                                                                                                |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, complying fully with content policies and containing no illegal or protected data. All processed data is legal and publicly available.