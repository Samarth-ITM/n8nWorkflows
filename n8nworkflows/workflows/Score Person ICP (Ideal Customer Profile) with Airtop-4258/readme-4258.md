Score Person ICP (Ideal Customer Profile) with Airtop

https://n8nworkflows.xyz/workflows/score-person-icp--ideal-customer-profile--with-airtop-4258


# Score Person ICP (Ideal Customer Profile) with Airtop

### 1. Workflow Overview

This workflow, titled **"Person ICP Scoring with Airtop"**, is designed to evaluate LinkedIn profiles against a defined Ideal Customer Profile (ICP) by leveraging the Airtop enrichment tool. It targets use cases where sales, marketing, or recruitment teams want to prioritize individuals based on their AI interest, seniority level, and technical expertise. The workflow automates data extraction from LinkedIn profiles and calculates a composite ICP score for assessing lead quality or fit.

The workflow is logically organized into four main blocks:

- **1.1 Input Reception:** Receives input data either via a web form submission or from another workflow trigger.
- **1.2 Parameter Preparation:** Normalizes and assigns variables for consistent downstream processing.
- **1.3 AI Processing & ICP Scoring:** Uses the Airtop node to extract structured profile data and calculate the ICP score based on custom scoring logic embedded in the prompt.
- **1.4 Output Formatting:** Cleans and formats the enriched data for downstream consumption or return.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Accepts LinkedIn profile URL and Airtop profile identifier either from a form submission or invoked by another workflow.

- **Nodes Involved:**  
  - On form submission (Form Trigger node)  
  - When Executed by Another Workflow (Execute Workflow Trigger node)

- **Node Details:**  

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Captures HTTP requests from an n8n-hosted form.  
    - *Configuration:* Form titled "ICP Scoring" with two required fields: "Linkedin Person Profile URL" and "Airtop Profile (connected to Linkedin)".  
    - *Inputs:* External HTTP calls (form submissions).  
    - *Outputs:* JSON containing the two form fields under keys matching the field labels.  
    - *Edge cases:* Missing required fields will prevent trigger; malformed URLs may cause downstream failures.

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Allows invocation of this workflow from other workflows with parameters.  
    - *Configuration:* Defines expected input parameters "Linkedin_URL" and "Airtop_profile".  
    - *Inputs:* Parameters passed from external workflow executions.  
    - *Outputs:* JSON object containing these parameters.  
    - *Edge cases:* Missing parameters or incorrect naming conventions may cause failures.

#### 2.2 Parameter Preparation

- **Overview:**  
Standardizes input parameter names from either trigger source into uniform variables for the Airtop node.

- **Nodes Involved:**  
  - Parameters (Set node)

- **Node Details:**  

  - **Parameters**  
    - *Type:* Set  
    - *Role:* Assigns consistent field names (`Linkedi_URL` and `Airtop_profile`) regardless of input source.  
    - *Configuration:* Uses expressions to map either form field names or workflow input parameters:  
      - `Linkedi_URL = {{$json["Linkedin Person Profile URL"] || $json.Linkedin_URL}}`  
      - `Airtop_profile = {{$json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile}}`  
    - *Inputs:* Output from either form trigger or workflow trigger nodes.  
    - *Outputs:* JSON with normalized fields `Linkedi_URL` and `Airtop_profile`.  
    - *Edge cases:* If both sources fail to provide values, subsequent nodes receive undefined variables.

#### 2.3 AI Processing & ICP Scoring

- **Overview:**  
Extracts detailed LinkedIn profile data and computes an ICP score using the Airtop enrichment API with a custom prompt describing the scoring logic.

- **Nodes Involved:**  
  - Calculate ICP PersonScoring (Airtop node)

- **Node Details:**  

  - **Calculate ICP PersonScoring**  
    - *Type:* Airtop (custom API integration node)  
    - *Role:* Queries Airtop's extraction resource with a detailed prompt to parse LinkedIn profile data and calculate ICP scores.  
    - *Configuration:*  
      - `url` parameter dynamically set to `{{$json.Linkedi_URL}}` — LinkedIn profile URL.  
      - `prompt` contains detailed extraction and scoring instructions specifying extraction of name, job title, employer, location, connections, followers, about text, AI interest level, seniority, technical depth, and the final ICP score based on weighted points.  
      - `resource:` "extraction"  
      - `operation:` "query"  
      - `profileName:` dynamically set to `{{$json.Airtop_profile}}` — authenticated Airtop profile linked to LinkedIn.  
      - `sessionMode:` "new" for fresh context per request.  
      - `outputSchema:` JSON schema enforcing strict typing and required fields for all extracted and scored data fields.  
    - *Inputs:* Normalized parameters from the "Parameters" node.  
    - *Outputs:* Response under `$json.data.modelResponse` containing structured JSON with extracted profile data and ICP score.  
    - *Version Requirements:* Airtop API credentials must be configured in n8n with valid access rights.  
    - *Edge Cases:*  
      - API authentication errors if Airtop credentials are missing or invalid.  
      - Timeout or rate limits if Airtop service is unavailable or limits exceeded.  
      - Parsing errors if LinkedIn profile URL is invalid or Airtop fails to extract data.  
    - *Retry Policy:* Enabled with max 2 tries to mitigate transient failures.

#### 2.4 Output Formatting

- **Overview:**  
Extracts and reformats the Airtop node response for clean output, making it ready for downstream use or return to the requester.

- **Nodes Involved:**  
  - Edit Fields (Set node)

- **Node Details:**  

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Converts nested response from Airtop into direct JSON output.  
    - *Configuration:*  
      - `jsonOutput` set to `={{ $json.data.modelResponse }}` to flatten the response.  
      - Mode set to "raw" to overwrite output with the extracted JSON.  
    - *Inputs:* Output from the Airtop node "Calculate ICP PersonScoring".  
    - *Outputs:* JSON object with all extracted fields and ICP score at the root level.  
    - *Edge Cases:* If the Airtop response is missing or malformed, this node outputs undefined or empty data.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                               | Input Node(s)                        | Output Node(s)                 | Sticky Note                                                                                          |
|-------------------------------|--------------------------------|----------------------------------------------|------------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                   | Receives input from user via web form        | —                                  | Parameters                    | ## Input parameters<br>* Linkedin Person Profile URL<br>* Airtop profile                           |
| When Executed by Another Workflow | Execute Workflow Trigger     | Receives input when triggered by another workflow | —                               | Parameters                    | ## Input parameters<br>* Linkedin Person Profile URL<br>* Airtop profile                           |
| Parameters                    | Set                           | Normalizes input parameters across triggers  | On form submission, When Executed by Another Workflow | Calculate ICP PersonScoring |                                                                                                    |
| Calculate ICP PersonScoring   | Airtop                        | Extracts LinkedIn data and calculates ICP score | Parameters                      | Edit Fields                  | ## Calculate ICP                                                                                   |
| Edit Fields                  | Set                           | Formats Airtop response for output            | Calculate ICP PersonScoring        | —                             | ## Calculate ICP                                                                                   |
| Sticky Note                  | Sticky Note                   | Documentation (Input parameters info)         | —                                  | —                             | ## Input parameters<br>* Linkedin Person Profile URL<br>* Airtop profile                           |
| Sticky Note1                 | Sticky Note                   | Documentation (ICP calculation focus)          | —                                  | —                             | ## Calculate ICP                                                                                   |
| Sticky Note2                 | Sticky Note                   | README style documentation with use case, method, setup, next steps | —                                  | —                             | README<br># Scoring LinkedIn Profiles Against Your ICP<br>Use Case, How It Works, Setup Requirements, Next Steps |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configure webhook with form titled "ICP Scoring"  
   - Add two required fields:  
     - "Linkedin Person Profile URL" (string)  
     - "Airtop Profile (connected to Linkedin)" (string)  
   - Save.

2. **Create Execute Workflow Trigger node**  
   - Name: `When Executed by Another Workflow`  
   - Type: Execute Workflow Trigger  
   - Define workflow inputs:  
     - `Linkedin_URL` (string)  
     - `Airtop_profile` (string)  
   - Save.

3. **Create Set node for parameter normalization**  
   - Name: `Parameters`  
   - Type: Set  
   - Add two variables:  
     - `Linkedi_URL` with expression:  
       `{{$json["Linkedin Person Profile URL"] || $json.Linkedin_URL}}`  
     - `Airtop_profile` with expression:  
       `{{$json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile}}`  
   - Connect both triggers (`On form submission` and `When Executed by Another Workflow`) to this node.

4. **Create Airtop node for extraction and scoring**  
   - Name: `Calculate ICP PersonScoring`  
   - Type: Airtop (custom integration node)  
   - Credentials: Select or create Airtop API credential with valid API key linked to your Airtop organization.  
   - Parameters:  
     - `url`: set expression to `{{$json.Linkedi_URL}}`  
     - `prompt`: paste the detailed prompt describing extraction fields and scoring logic (see overview section).  
     - `resource`: "extraction"  
     - `operation`: "query"  
     - `profileName`: set expression to `{{$json.Airtop_profile}}`  
     - `sessionMode`: "new"  
     - `additionalFields.outputSchema`: paste the JSON schema defining expected output fields and types.  
   - Enable retry on fail with max tries 2.  
   - Connect `Parameters` node output to this node.

5. **Create Set node to format output**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Configure:  
     - Mode: Raw  
     - JSON Output expression: `{{$json.data.modelResponse}}`  
   - Connect `Calculate ICP PersonScoring` node output to this node.

6. **Optional: Add Sticky Notes**  
   - Add sticky notes to document input parameters, ICP calculation, and overall README notes for clarity.

7. **Activate the workflow** and test with valid LinkedIn profile URLs and Airtop profile identifiers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| README content explains the use case, input parameters, scoring methodology, setup requirements, and possible next steps for embedding or batch processing. It emphasizes the importance of setting the ICP scoring method in the Airtop node prompt and having Airtop profiles connected to LinkedIn.                                                                                                         | Embedded in Sticky Note2 within the workflow                                                                |
| Airtop Profile connection must be established via the Airtop portal at https://portal.airtop.ai/browser-profiles to enable authorized extraction of LinkedIn profiles.                                                                                                                                                                                                                                   | https://portal.airtop.ai/browser-profiles                                                                    |
| Scoring system weights AI interest, technical depth, and seniority level differently, allowing customization for business priorities.                                                                                                                                                                                                                                                                      | Described in the prompt block of the Airtop node and README notes                                            |
| To avoid errors, ensure Airtop API credentials are properly configured in n8n's credential manager with necessary permissions.                                                                                                                                                                                                                                                                               | Workflow credential requirement                                                                              |
| The Airtop node uses a detailed prompt to instruct AI extraction; modifying scoring logic requires adjusting this prompt carefully while respecting the JSON schema for output.                                                                                                                                                                                                                              | Prompt inside the Airtop node's configuration                                                                |

---

**Disclaimer:** The content here is generated exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal or offensive material. All handled data is legal and public.