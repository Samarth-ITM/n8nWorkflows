Score LinkedIn Profiles Against Your ICP with Airtop

https://n8nworkflows.xyz/workflows/score-linkedin-profiles-against-your-icp-with-airtop-4205


# Score LinkedIn Profiles Against Your ICP with Airtop

### 1. Workflow Overview

This workflow, titled **"Score LinkedIn Profiles Against Your ICP with Airtop"**, automates the evaluation of individual LinkedIn profiles against a predefined Ideal Customer Profile (ICP). It is designed primarily for sales, recruiting, or marketing teams seeking to prioritize leads or candidates based on AI interest, technical expertise, and seniority.

**Target Use Cases:**

- Scoring and enriching LinkedIn profiles with detailed professional data.
- Prioritizing leads or candidates based on ICP alignment.
- Integration into CRM or lead management systems for automated segmentation.
- Batch processing profiles for sales or recruitment campaigns.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts input data either via a web form or through execution by another workflow.
- **1.2 Parameter Assignment:** Normalizes and sets input parameters for downstream processing.
- **1.3 ICP Person Scoring via Airtop:** Uses an Airtop node to query and extract detailed profile data and calculate the ICP score.
- **1.4 Output Formatting:** Cleans and formats the output JSON for downstream use.
- **1.5 Documentation and Guidance:** Sticky notes provide detailed descriptions, instructions, and context about the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block is responsible for receiving input parameters. It supports two triggers: a form submission trigger for manual input and a workflow execution trigger for automated input from other workflows.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Captures user input from a web form titled "ICP Scoring".  
  - *Configuration:*  
    - Two required fields: "Linkedin Person Profile URL" and "Airtop Profile (connected to Linkedin)".  
    - Webhook ID configured for external form submission.  
  - *Connections:* Outputs to "Parameters" node.  
  - *Edge Cases:* Missing or invalid URLs could cause downstream failures. Form validation is basic; deeper validation is recommended externally.  

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Accepts parameters from another workflow execution.  
  - *Configuration:* Defines expected workflow inputs "Linkedin_URL" and "Airtop_profile".  
  - *Connections:* Outputs to "Parameters" node.  
  - *Edge Cases:* Downstream nodes expect these inputs; missing or malformed inputs may cause failures.  

---

#### 1.2 Parameter Assignment

**Overview:**  
This block standardizes and assigns input parameters from either trigger to consistent variable names for use in the scoring node.

**Nodes Involved:**  
- Parameters

**Node Details:**

- **Parameters**  
  - *Type:* Set Node  
  - *Role:* Assigns variables "Linkedi_URL" and "Airtop_profile" by extracting from either form or workflow inputs, prioritizing form fields but falling back to workflow inputs.  
  - *Key Expressions:*  
    - `={{ $json["Linkedin Person Profile URL"] || $json.Linkedin_URL }}` for LinkedIn URL  
    - `={{ $json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile }}` for Airtop profile  
  - *Connections:* Outputs to "Calculate ICP PersonScoring".  
  - *Edge Cases:* If inputs are missing or incorrectly named, values may be undefined or null, leading to errors downstream.  

---

#### 1.3 ICP Person Scoring via Airtop

**Overview:**  
This block queries the Airtop API to extract detailed LinkedIn profile data and calculate an ICP score based on AI interest, technical depth, and seniority, using a detailed prompt.

**Nodes Involved:**  
- Calculate ICP PersonScoring

**Node Details:**

- **Calculate ICP PersonScoring**  
  - *Type:* Airtop Node (custom integration)  
  - *Role:* Calls Airtop API to:  
    - Extract fields such as full name, current job title, employer, LinkedIn company URL, location, connections, followers, about section text.  
    - Assess interest level in AI, seniority level, technical depth.  
    - Calculate ICP score based on weighted criteria provided in the prompt.  
  - *Configuration Highlights:*  
    - `url` parameter set dynamically to the LinkedIn URL (`={{ $json.Linkedi_URL }}`)  
    - `prompt` includes explicit instructions and scoring schema for Airtop's AI model.  
    - `profileName` set dynamically to Airtop profile (`={{ $json.Airtop_profile }}`)  
    - Session mode is "new" to ensure fresh context.  
    - Output schema strictly defines expected JSON fields and types for validation.  
    - `maxTries` set to 2 for retry on failure.  
  - *Connections:* Outputs to "Edit Fields" node.  
  - *Edge Cases:*  
    - Network or API authentication failures with Airtop credentials.  
    - Timeout if LinkedIn page is slow or inaccessible.  
    - Extraction inaccuracies if LinkedIn page format changes or is private.  
    - Expression errors if inputs are missing.  
  - *Credentials:* Airtop API credentials required and must be correctly configured.

---

#### 1.4 Output Formatting

**Overview:**  
This block refines Airtop's raw response to extract the relevant data model for downstream consumption.

**Nodes Involved:**  
- Edit Fields

**Node Details:**

- **Edit Fields**  
  - *Type:* Set Node  
  - *Role:* Overrides output to return only the JSON content under `data.modelResponse`, which contains the enriched profile and ICP score.  
  - *Configuration:*  
    - Mode set to raw JSON output with expression `={{ $json.data.modelResponse }}`  
  - *Connections:* Final output node (could be connected to external systems).  
  - *Edge Cases:* If `data.modelResponse` is missing or malformed, output will be empty or invalid.

---

#### 1.5 Documentation and Guidance

**Overview:**  
Sticky notes provide contextual information, instructions, and project documentation directly within the workflow canvas.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Sticky Note**  
  - Content: Lists input parameters ("Linkedin Person Profile URL", "Airtop profile").  
  - Purpose: Clarifies what input is expected at the start of the workflow.

- **Sticky Note1**  
  - Content: "Calculate ICP" indicating the core processing block.  
  - Purpose: Marks the main scoring step for clarity.

- **Sticky Note2**  
  - Content: Detailed README-style explanation covering use case, input/output, scoring methodology, setup instructions, and next steps.  
  - Useful Links: https://portal.airtop.ai/browser-profiles  
  - Purpose: Comprehensive project documentation embedded for maintainers.  
  - Size: Large note, covers broad context.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                   | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                                        |
|----------------------------|------------------------------|---------------------------------|-----------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                 | Input reception via web form    | —                           | Parameters               | This automation takes person's Linkedin Profile URL and Airtop Profile (authenticated for Linkedin) and returns the person's ICP score |
| When Executed by Another Workflow | Execute Workflow Trigger    | Input reception via workflow    | —                           | Parameters               | —                                                                                                                                 |
| Parameters                | Set                          | Assign normalized variables      | On form submission, When Executed by Another Workflow | Calculate ICP PersonScoring | —                                                                                                                                 |
| Calculate ICP PersonScoring| Airtop Node                  | Extract LinkedIn data, score ICP| Parameters                  | Edit Fields              | ## Calculate ICP                                                                                                                  |
| Edit Fields               | Set                          | Format and clean output          | Calculate ICP PersonScoring | —                        | —                                                                                                                                 |
| Sticky Note               | Sticky Note                  | Documentation                   | —                           | —                        | ## Input parameters * Linkedin Person Profile URL * Airtop profile                                                                |
| Sticky Note1              | Sticky Note                  | Documentation                   | —                           | —                        | ## Calculate ICP                                                                                                                  |
| Sticky Note2              | Sticky Note                  | Documentation                   | —                           | —                        | README with detailed use case, instructions, scoring method, setup, and next steps. See https://portal.airtop.ai/browser-profiles |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**
   - Type: Form Trigger  
   - Set Form Title: "ICP Scoring"  
   - Add two required fields:  
     - "Linkedin Person Profile URL" (string, required)  
     - "Airtop Profile (connected to Linkedin)" (string, required)  
   - Configure webhook for external form submission.

2. **Create an Execute Workflow Trigger Node:**
   - Type: Execute Workflow Trigger  
   - Define inputs: "Linkedin_URL" and "Airtop_profile" (both strings).  
   - This allows triggering from other workflows.

3. **Create a Set Node named "Parameters":**
   - Assign two variables:  
     - `Linkedi_URL` = `{{$json["Linkedin Person Profile URL"] || $json.Linkedin_URL}}`  
     - `Airtop_profile` = `{{$json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile}}`  
   - Connect outputs of both triggers ("On form submission" and "When Executed by Another Workflow") to this node.

4. **Create Airtop Node named "Calculate ICP PersonScoring":**
   - Set resource: "extraction", operation: "query".  
   - In parameters:  
     - `url`: `={{ $json.Linkedi_URL }}`  
     - `prompt`: Paste the detailed prompt requesting extraction of LinkedIn data fields and scoring instructions (as per original).  
     - `profileName`: `={{ $json.Airtop_profile }}`  
     - `sessionMode`: "new"  
     - `outputSchema`: Use the provided JSON schema defining required fields (full name, titles, scores, etc.).  
   - Configure retry on failure (2 tries).  
   - Connect "Parameters" node output to this node.

5. **Create Set Node named "Edit Fields":**
   - Set mode to raw JSON output.  
   - Expression for JSON output: `={{ $json.data.modelResponse }}`  
   - Connect output of "Calculate ICP PersonScoring" node to this node.

6. **Add Sticky Notes for documentation:**
   - Sticky Note 1: Input parameters ("Linkedin Person Profile URL" and "Airtop profile").  
   - Sticky Note 2: Label "Calculate ICP" near the scoring node.  
   - Sticky Note 3: Detailed README including use case, input/output, scoring rules, setup instructions, and next steps. Include link to Airtop profiles portal: https://portal.airtop.ai/browser-profiles.

7. **Credentials Setup:**
   - Configure Airtop API credentials in n8n under the credential "Airtop Official Org".  
   - Ensure the Airtop profile used is connected and authenticated with LinkedIn.

8. **Connections:**
   - "On form submission" → "Parameters"  
   - "When Executed by Another Workflow" → "Parameters"  
   - "Parameters" → "Calculate ICP PersonScoring"  
   - "Calculate ICP PersonScoring" → "Edit Fields"  

9. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This automation scores LinkedIn profiles against an Ideal Customer Profile (ICP) by analyzing AI interest, technical depth, and seniority level.                                                                                   | General project description                       |
| ICP scoring uses weighted criteria: AI Interest (5-35 pts), Technical Depth (5-35 pts), Seniority Level (5-30 pts) for a total out of 100.                                                                                          | Embedded in Airtop node prompt                    |
| Requires Airtop profile connected to LinkedIn and Airtop API credentials configured in n8n.                                                                                                                                          | Setup instructions                                |
| Useful resource for Airtop profiles: https://portal.airtop.ai/browser-profiles                                                                                                                                                       | Official Airtop portal                            |
| The workflow supports two input methods: form submission for manual use and workflow trigger for automation and integration.                                                                                                        | Integration flexibility                           |
| Potential failure points include missing inputs, Airtop API issues, and LinkedIn page accessibility or formatting changes.                                                                                                         | Error handling notes                              |
| The workflow can be embedded into CRM systems for automated lead scoring or batch processed for large-scale lead prioritization.                                                                                                   | Next steps and integration ideas                  |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively from an n8n automation workflow. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.