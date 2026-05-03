Extract Structured LinkedIn Profile Data with Airtop & AI Parsing

https://n8nworkflows.xyz/workflows/extract-structured-linkedin-profile-data-with-airtop---ai-parsing-4203


# Extract Structured LinkedIn Profile Data with Airtop & AI Parsing

### 1. Workflow Overview

This workflow automates the extraction of detailed, structured data from public LinkedIn profiles using Airtop’s browser automation combined with AI parsing. It targets users who need reliable, fast, and error-minimized LinkedIn data extraction for use cases such as lead enrichment, hiring research, and contact profiling. The workflow accepts LinkedIn profile URLs and corresponding Airtop browser profile names as inputs, then opens the LinkedIn profile in a real browser session, runs an AI-powered extraction prompt, and outputs a rich JSON object containing key profile information.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Accepts inputs either via a web form submission or from another workflow.
- **1.2 Parameter Preparation:** Normalizes and sets the LinkedIn URL and Airtop profile name into workflow variables.
- **1.3 Data Extraction via Airtop:** Uses the Airtop node to open the LinkedIn profile and extract structured data with a detailed AI prompt.
- **1.4 Data Formatting:** Extracts the relevant JSON response from Airtop's output for downstream use.
- **1.5 Documentation:** A large sticky note provides comprehensive documentation and setup instructions for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Handles the entry points for the workflow. It can either start from a web form submission (ideal for manual triggering) or from execution by another workflow (ideal for automation chains), both providing the LinkedIn URL and Airtop profile name.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point triggered when a user submits a web form titled "Linkedin Profile Extractor" with two required fields: "Linkedin Person Profile URL" and "Airtop Profile (connected to Linkedin)".  
  - Configuration: Captures the form inputs as JSON for use downstream.  
  - Connections: Outputs directly feed into "Parameters".  
  - Edge Cases: Invalid or missing form inputs will cause workflow failure or no data extraction. The webhook must be publicly accessible.  
  - Version: 2.2  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered programmatically from another workflow, accepting inputs "Linkedin_URL" and "Airtop_profile".  
  - Configuration: Expects these two parameters in the input JSON.  
  - Connections: Outputs feed into "Parameters".  
  - Edge Cases: Missing or malformed input parameters can cause errors.  
  - Version: 1.1  

---

#### 2.2 Parameter Preparation

**Overview:**  
Consolidates input data by setting variables for the LinkedIn URL and Airtop profile name from either the form or workflow inputs, ensuring consistency for downstream nodes.

**Nodes Involved:**  
- Parameters

**Node Details:**  

- **Parameters**  
  - Type: Set  
  - Role: Normalizes and assigns the two critical parameters:  
    - `Linkedi_URL`: Takes from form field "Linkedin Person Profile URL" or from "Linkedin_URL" input, prioritizing form input if available.  
    - `Airtop_profile`: Takes from form field "Airtop Profile (connected to Linkedin)" or from "Airtop_profile" input.  
  - Expressions: Uses conditional expressions to select the appropriate source for each variable.  
  - Inputs: From either "On form submission" or "When Executed by Another Workflow".  
  - Outputs: Feeds into "Airtop" node.  
  - Edge Cases: If neither input source provides the URL or profile, subsequent nodes will fail.  
  - Version: 3.4  

---

#### 2.3 Data Extraction via Airtop

**Overview:**  
This key block uses the Airtop node to launch a real browser session with the specified Airtop profile, navigate to the LinkedIn URL, and extract structured data using an AI prompt tailored for LinkedIn profiles.

**Nodes Involved:**  
- Airtop

**Node Details:**  

- **Airtop**  
  - Type: Airtop (n8n community node)  
  - Role: Automates a browser session to fetch and parse LinkedIn profile data.  
  - Configuration:  
    - URL: Uses the variable `Linkedi_URL` from Parameters node.  
    - Profile Name: Uses variable `Airtop_profile` for the browser profile connected to LinkedIn.  
    - Prompt: A multi-line AI prompt instructing extraction of detailed profile elements including name, headline, location, experience, education, skills, certifications, languages, connections, and recommendations.  
    - Session Mode: New session per execution.  
    - Output Schema: JSON Schema strictly defining expected fields and nested structures, ensuring data is validated and structured.  
  - Credentials: Uses "Airtop Official Org" API key credential.  
  - Inputs: Connected from "Parameters" node.  
  - Outputs: Provides a JSON object under `data.modelResponse`.  
  - Edge Cases:  
    - Network errors or Airtop API failures.  
    - Invalid or inaccessible LinkedIn URLs.  
    - Airtop profile not logged in or expired sessions causing authentication failures.  
    - AI prompt parsing errors or unexpected LinkedIn profile layout changes.  
  - Version: 1  

---

#### 2.4 Data Formatting

**Overview:**  
Processes the raw output from Airtop node, extracting the parsed model response JSON for downstream usage or storage.

**Nodes Involved:**  
- Edit Fields

**Node Details:**  

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts the nested JSON `data.modelResponse` from Airtop's output and sets it as the new JSON output for the workflow.  
  - Configuration: Uses expression `={{ $json.data.modelResponse }}` to directly assign the parsed data.  
  - Input: Connected from "Airtop" node.  
  - Outputs: Final structured JSON payload ready for further automation or storage.  
  - Edge Cases: If Airtop did not return the expected structure, this node may output null or error.  
  - Version: 3.4  

---

#### 2.5 Documentation and Notes

**Overview:**  
Contains a large sticky note providing detailed workflow documentation, use cases, setup instructions, and pointers for extensions.

**Nodes Involved:**  
- Sticky Note

**Node Details:**  

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Provides comprehensive documentation inside n8n editor for users and maintainers.  
  - Content:  
    - Explains the purpose: fast and accurate LinkedIn profile extraction.  
    - Describes inputs: LinkedIn URL and Airtop profile.  
    - Details workflow steps and AI prompt contents.  
    - Lists setup requirements including Airtop credentials and profile setup.  
    - Suggests next steps such as CRM integration or bulk processing.  
    - Links to Airtop profiles and API key setup portals.  
  - Position: Off to the top-left for visibility.  
  - Edge Cases: None (informational only).  

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                       | Input Node(s)                   | Output Node(s)             | Sticky Note                                                                                                                     |
|-----------------------------|-----------------------------|------------------------------------|--------------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| On form submission           | Form Trigger                | Entry point via user web form       | None                           | Parameters                 | This Airtop Studio automation simplifies LinkedIn data extraction by automatically providing structured, reliable data.       |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point via workflow call       | None                           | Parameters                 | This Airtop Studio automation simplifies LinkedIn data extraction by automatically providing structured, reliable data.       |
| Parameters                  | Set                         | Normalizes and sets input variables | On form submission, When Executed by Another Workflow | Airtop                     |                                                                                                                                 |
| Airtop                      | Airtop                      | Opens LinkedIn profile, extracts data | Parameters                     | Edit Fields                | Uses Airtop API and browser profile connected to LinkedIn for AI-based structured data extraction.                             |
| Edit Fields                 | Set                         | Extracts parsed JSON from Airtop output | Airtop                       | None                       |                                                                                                                                 |
| Sticky Note                 | Sticky Note                 | Documentation and instructions     | None                           | None                       | Comprehensive README and setup instructions inside n8n editor.                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**  
   - Type: Form Trigger  
   - Configure form title: "Linkedin Profile Extractor"  
   - Add two required fields:  
     - "Linkedin Person Profile URL" (string)  
     - "Airtop Profile (connected to Linkedin)" (string)  
   - Save node.

2. **Create the "When Executed by Another Workflow" node:**  
   - Type: Execute Workflow Trigger  
   - Configure inputs:  
     - "Linkedin_URL"  
     - "Airtop_profile"  
   - Save node.

3. **Create the "Parameters" node:**  
   - Type: Set  
   - Add two string variables:  
     - `Linkedi_URL` with expression: `={{ $json["Linkedin Person Profile URL"] || $json.Linkedin_URL }}`  
     - `Airtop_profile` with expression: `={{ $json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile }}`  
   - Save node.

4. **Connect outputs:**  
   - Connect "On form submission" main output to "Parameters" input.  
   - Connect "When Executed by Another Workflow" main output to "Parameters" input.

5. **Create the "Airtop" node:**  
   - Type: Airtop  
   - Set parameters:  
     - URL: `={{ $json.Linkedi_URL }}`  
     - Profile Name: `={{ $json.Airtop_profile }}`  
     - Prompt: Paste the detailed AI prompt for LinkedIn extraction (include all fields as in original workflow).  
     - Operation: Query  
     - Resource: Extraction  
     - Session Mode: New  
     - Output Schema: Paste the full JSON schema defining expected output structure.  
   - Credentials: Select or create Airtop API credentials ("Airtop Official Org").  
   - Save node.

6. **Connect "Parameters" node output to "Airtop" node input.**

7. **Create the "Edit Fields" node:**  
   - Type: Set  
   - Configure to set JSON output as: `={{ $json.data.modelResponse }}`  
   - Save node.

8. **Connect "Airtop" node main output to "Edit Fields" node input.**

9. **Add a "Sticky Note" node:**  
   - Type: Sticky Note  
   - Paste the detailed README content describing workflow purpose, inputs, setup, and usage.  
   - Position the note for clarity.

10. **Activate the workflow and test:**  
    - Submit the form or trigger via another workflow with valid LinkedIn URL and Airtop profile name.  
    - Verify structured JSON output in "Edit Fields" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| This automation requires an Airtop API key and a browser profile connected to LinkedIn for authenticated scraping.                                                                                                                                | https://portal.airtop.ai/api-keys and https://portal.airtop.ai/browser-profiles |
| Output JSON schema ensures data consistency and validation, facilitating integration with CRMs or databases.                                                                                                                                       | JSON Schema Draft-07                                        |
| Recommended next steps include integrating with CRM systems (HubSpot, Salesforce, Airtable) or combining with LinkedIn search scrapers for bulk profile processing.                                                                              | -                                                           |
| The workflow is designed to be extensible by adapting the AI prompt to other platforms like GitHub, Twitter, or company websites.                                                                                                               | -                                                           |
| The sticky note inside the workflow provides extensive documentation to aid users and maintainers in understanding and extending the automation.                                                                                                | -                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.