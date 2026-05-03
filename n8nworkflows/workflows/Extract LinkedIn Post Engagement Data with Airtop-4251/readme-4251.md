Extract LinkedIn Post Engagement Data with Airtop

https://n8nworkflows.xyz/workflows/extract-linkedin-post-engagement-data-with-airtop-4251


# Extract LinkedIn Post Engagement Data with Airtop

### 1. Workflow Overview

This workflow is designed to extract detailed engagement data from a given LinkedIn post URL using an Airtop AI agent. It targets use cases such as marketing analysis, lead generation, content performance measurement, and research by gathering quantitative and qualitative insights about post interactions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Handles trigger events from either an external workflow or a form submission, receiving the LinkedIn post URL and Airtop profile details.
- **1.2 Data Preparation**: Maps and normalizes input fields for consistent downstream processing.
- **1.3 AI Processing with Airtop**: Uses the Airtop node to open a browser session, loads the LinkedIn post, and runs an AI prompt to extract engagement metrics and user details.
- **1.4 Response Parsing**: Parses the JSON response from Airtop into structured output data.
- **1.5 Documentation and User Guidance**: Provides extensive sticky notes explaining workflow purpose, inputs, outputs, and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block waits for input data to start the workflow, supporting two entry points: manual form submission or execution from another workflow.
- **Nodes Involved:**  
  - On form submission  
  - When Executed by Another Workflow

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts workflow upon user form submission via webhook.  
    - *Configuration:*  
      - Fields: "Airtop Profile (Logged into Linkedin)" (required), "Post URL" (required)  
      - Form title and description provide user guidance on inputs and purpose.  
    - *Inputs:* External user form input  
    - *Outputs:* JSON object with form inputs  
    - *Potential Failures:* Webhook errors, missing required fields, malformed URLs.

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Allows this workflow to be called from other workflows with predefined inputs.  
    - *Configuration:* Defines two required inputs: airtop_profile and linkedin_post_url.  
    - *Inputs:* JSON parameters from calling workflow  
    - *Outputs:* JSON object with inputs for further processing  
    - *Potential Failures:* Missing input parameters, workflow triggering errors.

#### 2.2 Data Preparation

- **Overview:** Normalizes and maps incoming inputs to standard variable names to unify data format regardless of trigger source.
- **Nodes Involved:**  
  - Map fields

- **Node Details:**

  - **Map fields**  
    - *Type:* Set Node  
    - *Role:* Assigns or remaps input variables `airtop_profile` and `post_url` from either the form or workflow trigger inputs.  
    - *Configuration:*  
      - airtop_profile: From `$json.airtop_profile` or form field "Airtop Profile (Logged into Linkedin)"  
      - post_url: From `$json.linkedin_post_url` or form field "Post URL"  
    - *Inputs:* JSON from trigger nodes  
    - *Outputs:* JSON with normalized keys `airtop_profile` and `post_url`  
    - *Edge Cases:* Missing or empty inputs, inconsistent naming conventions.

#### 2.3 AI Processing with Airtop

- **Overview:** Uses Airtop's AI browser automation to load the LinkedIn post URL and extract structured engagement data based on a custom prompt.
- **Nodes Involved:**  
  - Airtop

- **Node Details:**

  - **Airtop**  
    - *Type:* Airtop Node (Browser Automation + AI agent)  
    - *Role:* Opens a new browser session with the specified Airtop profile, loads the LinkedIn post URL, and runs a prompt instructing the AI to extract engagement data.  
    - *Configuration:*  
      - URL: Taken dynamically from `post_url` variable  
      - Prompt: Custom prompt requesting extraction of:  
        - Names, job titles, profile URLs of commenters/reactors  
        - Total counts of reactions, comments, reposts  
        - Defaults counts to -1 if unavailable  
      - Resource: extraction  
      - Operation: query  
      - Profile Name: dynamically set from `airtop_profile`  
      - Session Mode: new (starts a fresh browser session)  
      - Output Schema: JSON schema defining expected structured response including arrays of interactors and counts  
    - *Inputs:* JSON with `post_url` and `airtop_profile`  
    - *Outputs:* JSON string response containing the extracted data under `data.modelResponse`  
    - *Edge Cases:* Authentication failure if Airtop profile not logged in, network timeout, LinkedIn page structure changes causing extraction failure, prompt misunderstanding by AI agent.

#### 2.4 Response Parsing

- **Overview:** Converts the raw JSON string response from Airtop node into a structured JSON object for downstream use or output.
- **Nodes Involved:**  
  - Parse engagement analysis response

- **Node Details:**

  - **Parse engagement analysis response**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Parses the JSON string contained in `data.modelResponse` into a JSON object and outputs it.  
    - *Configuration:*  
      - JS Code: `const result = JSON.parse($input.first().json.data.modelResponse); return [{json: {...result}}];`  
    - *Inputs:* Output from Airtop node (raw JSON string)  
    - *Outputs:* Parsed JSON object with engagement data fields: interactors array, reactions_count, comments_count, reposts_count  
    - *Edge Cases:* Malformed JSON string, empty response, parsing exceptions.

#### 2.5 Documentation and User Guidance

- **Overview:** Provides contextual information, usage instructions, and detailed explanation of the workflow via sticky notes.
- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note

- **Node Details:**

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Role:* Summarizes the core extraction capabilities of the workflow.  
    - *Content:* Explains that the workflow extracts number of reactions, comments, reposts, and detailed info about commenters/reactors.

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Role:* Provides a comprehensive README-like explanation including use case, input parameters, workflow steps, and output format.  
    - *Content:*  
      - Use case description for marketing and research  
      - Detailed input parameter table specifying required fields  
      - Stepwise explanation of workflow operation  
      - Example JSON output schema  
      - Link to Airtop Profiles documentation: https://portal.airtop.ai/browser-profiles

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                             | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                           |
|-------------------------------|---------------------------|--------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger              | Start workflow via user form input          | N/A                              | Map fields                      |                                                                                                     |
| When Executed by Another Workflow | Execute Workflow Trigger | Start workflow via external workflow call   | N/A                              | Map fields                      |                                                                                                     |
| Map fields                    | Set Node                  | Normalize input variables                    | On form submission, When Executed by Another Workflow | Airtop                         |                                                                                                     |
| Airtop                       | Airtop Node               | AI browser automation and data extraction   | Map fields                      | Parse engagement analysis response |                                                                                                     |
| Parse engagement analysis response | Code Node               | Parse raw JSON response into structured data | Airtop                         | N/A                             |                                                                                                     |
| Sticky Note1                  | Sticky Note               | Workflow core functionality summary         | N/A                              | N/A                             | ## Extract engagement data With airtop and a simple prompt, the agent extracts: number comments, reactions, reposts, plus commenters details |
| Sticky Note                  | Sticky Note               | Detailed README and usage instructions       | N/A                              | N/A                             | README with use case, input/output details, and workflow steps. Link: https://portal.airtop.ai/browser-profiles |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Nodes**

   - Add a **Form Trigger** node named `On form submission`.  
     - Configure webhook ID (auto-generated).  
     - Set Form Title: "LinkedIn Post Engagement Data Extractor".  
     - Add required fields:  
       - "Airtop Profile (Logged into Linkedin)" (required)  
       - "Post URL" (required)  
     - Add descriptive form text explaining the use case.

   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
     - Define workflow inputs:  
       - airtop_profile  
       - linkedin_post_url

2. **Add a Set Node for Field Mapping**

   - Add a **Set** node named `Map fields`.  
   - Create two fields:  
     - `airtop_profile` with expression: `{{$json.airtop_profile || $json["Airtop Profile (Logged into Linkedin)"]}}`  
     - `post_url` with expression: `{{$json.linkedin_post_url || $json["Post URL"]}}`  
   - Connect both trigger nodes (`On form submission` and `When Executed by Another Workflow`) to this node.

3. **Add Airtop Node for AI Extraction**

   - Add an **Airtop** node named `Airtop`.  
   - Set parameters:  
     - URL: Expression `{{$json.post_url}}`  
     - Prompt:  
       ```
       You are looking at a LinkedIn post.
       Your job is to extract engagement data from the post.

       Extract the information of the people that have commented and reacted to this post. I want their name, their job title and their profile_url.

       I also want the total number of reactions, comments and reposts. For each of these values, if you are unable to find it, set it as -1.
       ```  
     - Resource: extraction  
     - Operation: query  
     - Profile Name: Expression `{{$json.airtop_profile}}`  
     - Session Mode: new  
     - Additional Fields → Output Schema: (copy the detailed JSON schema from the workflow JSON or define as below)  
       ```json
       {
         "type": "object",
         "properties": {
           "interactors": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "name": { "type": "string" },
                 "job_title": { "type": "string" },
                 "profile_url": { "type": "string" }
               },
               "required": ["name", "job_title"],
               "additionalProperties": false
             }
           },
           "reactions_count": { "type": "number" },
           "comments_count": { "type": "number" },
           "reposts_count": { "type": "number" }
         },
         "required": ["interactors", "reactions_count", "comments_count", "reposts_count"],
         "additionalProperties": false,
         "$schema": "http://json-schema.org/draft-07/schema#"
       }
       ```
   - Connect `Map fields` node to `Airtop`.

4. **Add Code Node to Parse Response**

   - Add a **Code** node named `Parse engagement analysis response`.  
   - Set language to JavaScript (Version 2).  
   - Use the code:  
     ```javascript
     const result = JSON.parse($input.first().json.data.modelResponse);
     return [{ json: { ...result } }];
     ```  
   - Connect `Airtop` node output to this node.

5. **(Optional) Add Sticky Notes**

   - Add sticky notes to document the workflow:  
     - One summarizing the workflow purpose and data extracted.  
     - Another with detailed README including use cases, input descriptions, workflow steps, and output format.

6. **Set Credentials**

   - For the Airtop node, assign valid Airtop credentials with a logged-in browser profile to LinkedIn.  
   - No other credentials are required.

7. **Save and Activate Workflow**

   - Save the entire workflow.  
   - Activate it to enable triggers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow uses Airtop AI browser automation which requires a valid Airtop profile logged into LinkedIn. Profiles can be managed at https://portal.airtop.ai/browser-profiles.                                                                                     | Airtop Profiles Management                           |
| The prompt embedded in the Airtop node is critical for correct data extraction; changes to LinkedIn’s UI or post structure may require prompt or schema updates.                                                                                                     | Prompt design and maintenance                        |
| Output data includes fallback values (-1) for missing metrics to handle cases where counts are not available on the LinkedIn post.                                                                                                                                    | Data robustness                                     |
| The workflow supports two types of triggers to enable flexible integrations: manual form input or triggering from other workflows.                                                                                                                                     | Trigger flexibility                                  |
| Sticky notes within the workflow provide detailed explanations and usage instructions. Review them to understand input requirements and output format.                                                                                                               | Documentation within workflow                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n integration and automation tool. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.