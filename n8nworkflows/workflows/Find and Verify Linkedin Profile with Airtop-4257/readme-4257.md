Find and Verify Linkedin Profile with Airtop

https://n8nworkflows.xyz/workflows/find-and-verify-linkedin-profile-with-airtop-4257


# Find and Verify Linkedin Profile with Airtop

### 1. Workflow Overview

This workflow automates the discovery and verification of LinkedIn profile URLs based on provided person information. It is targeted primarily at marketing, recruiting, and sales professionals who need to enrich contact data with verified LinkedIn profiles. The workflow accepts input either via a web form or from another workflow and optionally uses an Airtop authenticated LinkedIn browser profile to confirm the validity of the found LinkedIn URL.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts person info and optional Airtop profile via form submission or workflow trigger.
- **1.2 Parameter Unification:** Normalizes and unifies input parameters from different sources.
- **1.3 Profile URL Search:** Performs a Google search to find the most probable LinkedIn profile URL.
- **1.4 LinkedIn URL Validation:** Filters out invalid or missing URLs based on format and presence of Airtop profile.
- **1.5 Profile URL Verification:** Uses Airtop to verify if the found LinkedIn profile matches the person info provided.
- **1.6 Output Handling:** Returns the verified LinkedIn URL or "NA" if no valid profile is found.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures input data either from an external form submission or from another workflow execution trigger. It provides flexibility for manual use or integration into larger automated processes.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow  
- Sticky Note (Input Parameters)

**Node Details:**

- **On form submission**  
  - Type: `formTrigger`  
  - Role: Receives HTTP webhook requests when a user submits the web form.  
  - Configuration:  
    - Form titled "Linkedin Profile Extractor" with two required fields: "Person Info" and "Airtop Profile (connected to Linkedin)".  
    - Form description explains the purpose and usage, including a link to Airtop Profiles.  
  - Input: HTTP POST data from form submission.  
  - Output: JSON containing form fields.  
  - Edge cases: Missing required fields will block submission; malformed input could cause errors downstream.  
  - Notes: Provides entry point for manual user input.

- **When Executed by Another Workflow**  
  - Type: `executeWorkflowTrigger`  
  - Role: Enables this workflow to be triggered by another workflow with parameters.  
  - Configuration: Expects inputs "Person_info" and "Airtop_profile".  
  - Input: Workflow data structure.  
  - Output: JSON with input parameters forwarded downstream.  
  - Edge cases: Missing inputs from calling workflow; incompatible data types.

- **Sticky Note** (Input Parameters)  
  - Content: "Run this workflow using a form or from another workflow"  
  - Role: Documentation for users on how to trigger the workflow.

---

#### 2.2 Parameter Unification

**Overview:**  
Normalizes input parameters regardless of source, ensuring downstream nodes receive consistent data fields named "Person_info" and "Airtop_profile."

**Nodes Involved:**  
- Unify Parameters

**Node Details:**

- **Unify Parameters**  
  - Type: `set`  
  - Role: Sets unified output variables for consistent usage.  
  - Configuration:  
    - Assigns "Person_info" from either `Person Info` (form field) or `Person_info` (workflow input).  
    - Assigns "Airtop_profile" similarly from form or workflow input.  
  - Input: Raw JSON from form or workflow trigger.  
  - Output: JSON with standardized keys "Person_info" and "Airtop_profile."  
  - Expressions used:  
    - `={{ $json["Person Info"] || $json.Person_info }}`  
    - `={{ $json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile }}`  
  - Edge cases: Empty or missing input fields result in empty strings; downstream nodes must handle these gracefully.

---

#### 2.3 Profile URL Search

**Overview:**  
Uses Airtop’s extraction capabilities to perform a Google search for the LinkedIn profile URL based on the provided person info.

**Nodes Involved:**  
- Search Profile URL  
- Sticky Note1 (Search LinkedIn)

**Node Details:**

- **Search Profile URL**  
  - Type: `airtop` (custom node for Airtop API)  
  - Role: Queries Google search results and extracts the most likely LinkedIn URL.  
  - Configuration:  
    - URL: `https://www.google.com/search?q={{ encodeURI($json['Person_info']+' Linkedin') }}` to search Google for the person plus "Linkedin".  
    - Prompt instructs Airtop to extract only the most likely LinkedIn profile URL or return "NA" if none found.  
    - Resource: "extraction" with operation "query".  
    - Session mode: "new".  
    - Credential: Airtop API credentials with official organization.  
  - Input: Unified parameter "Person_info".  
  - Output: JSON containing `data.modelResponse` with extracted LinkedIn URL or "NA".  
  - Edge cases: Google search may fail or return no LinkedIn results; Airtop extraction may time out or misinterpret data; network errors.  
  - Notes: May take several minutes depending on query complexity.

- **Sticky Note1**  
  - Content: Explains the search and verification logic.  
  - Role: Documentation for workflow users.

---

#### 2.4 LinkedIn URL Validation

**Overview:**  
Filters the extracted LinkedIn URL to ensure it is valid, non-empty, contains "linkedin.com/in", and that an Airtop profile is provided before proceeding to verification.

**Nodes Involved:**  
- Is valid Linkedin link?

**Node Details:**

- **Is valid Linkedin link?**  
  - Type: `filter`  
  - Role: Applies conditions to decide if URL is valid for verification.  
  - Conditions:  
    - Airtop_profile is not empty (non-empty string).  
    - Extracted LinkedIn URL does not start with "NA".  
    - Extracted LinkedIn URL contains "linkedin.com/in".  
  - Input: Output from "Search Profile URL" and unified parameters.  
  - Output: Passes valid LinkedIn URLs to next node; filters out invalid URLs, skipping verification step.  
  - Edge cases: Empty Airtop profile skips verification; malformed URLs filtered out; false positives if URL format changes.

---

#### 2.5 Profile URL Verification

**Overview:**  
Uses Airtop authenticated LinkedIn browser profile to visit and analyze the found LinkedIn URL, verifying if it matches the person info provided.

**Nodes Involved:**  
- Verify Profile URL

**Node Details:**

- **Verify Profile URL**  
  - Type: `airtop`  
  - Role: Visits and analyzes the LinkedIn profile page to confirm if it belongs to the person.  
  - Configuration:  
    - URL: extracted LinkedIn URL from previous step.  
    - Prompt: Asks Airtop to analyze experience/history and confirm if the profile matches "Person_info".  
    - Returns the URL if valid, otherwise "NA".  
    - Uses Airtop profile name supplied for authenticated browsing.  
    - Session mode: "new".  
  - Input: Valid LinkedIn URLs passing filter and unified parameters (Airtop profile name).  
  - Output: Verified LinkedIn URL or "NA".  
  - Edge cases: Verification may fail due to profile privacy, session expiration, or mismatch; Airtop API errors; network timeouts.

---

#### 2.6 Output Handling

**Overview:**  
The workflow returns the verified LinkedIn profile URL or "NA" if no valid or verified profile is found. The endpoint for output is the termination of the verification node.

**Nodes Involved:**  
- No explicit output node; final output is the last node's data.

**Node Details:**

- The workflow's last node is "Verify Profile URL" that outputs the final verified URL or "NA".  
- If the "Is valid Linkedin link?" filter fails, no verification occurs and the workflow ends with the original extracted URL or "NA".  
- Downstream workflows or integrations can consume this output as needed.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                                  | Input Node(s)                      | Output Node(s)          | Sticky Note                                                                                              |
|---------------------------|---------------------------|-------------------------------------------------|----------------------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| On form submission        | formTrigger               | Receives user input via web form                 | None                             | Unify Parameters        | Run this workflow using a form or from another workflow                                                 |
| When Executed by Another Workflow | executeWorkflowTrigger | Receives input from another workflow             | None                             | Unify Parameters        | Run this workflow using a form or from another workflow                                                 |
| Unify Parameters          | set                       | Normalizes and unifies input parameters          | On form submission, When Executed by Another Workflow | Search Profile URL     | Run this workflow using a form or from another workflow                                                 |
| Search Profile URL        | airtop                    | Searches Google to extract probable LinkedIn URL | Unify Parameters                | Is valid Linkedin link? | Search for the LinkedIn profile using Google. If a link is found, verify that the profile belongs to the correct person. |
| Is valid Linkedin link?   | filter                    | Validates format and presence of LinkedIn URL   | Search Profile URL               | Verify Profile URL      | Search for the LinkedIn profile using Google. If a link is found, verify that the profile belongs to the correct person. |
| Verify Profile URL        | airtop                    | Verifies LinkedIn profile via Airtop profile    | Is valid Linkedin link?          | (Workflow output)       | Search for the LinkedIn profile using Google. If a link is found, verify that the profile belongs to the correct person. |
| Sticky Note               | stickyNote                | Documentation                                    | None                            | None                   | Run this workflow using a form or from another workflow                                                 |
| Sticky Note1              | stickyNote                | Documentation                                    | None                            | None                   | Search for the LinkedIn profile using Google. If a link is found, verify that the profile belongs to the correct person. |
| Sticky Note7              | stickyNote                | Full README and detailed project overview       | None                            | None                   | README content with use cases, setup, and next steps including https://portal.airtop.ai/browser-profiles |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**  
   - Type: `formTrigger`  
   - Configure form title: "Linkedin Profile Extractor"  
   - Add two required form fields:  
     - "Person Info" (text)  
     - "Airtop Profile (connected to Linkedin)" (text)  
   - Add form description with HTML content explaining usage and link to Airtop Profiles page: `https://portal.airtop.ai/browser-profiles`  
   - This node acts as a webhook entry point.

2. **Create the "When Executed by Another Workflow" node:**  
   - Type: `executeWorkflowTrigger`  
   - Configure to accept inputs: "Person_info" and "Airtop_profile"  
   - This node provides an alternative trigger method.

3. **Create the "Unify Parameters" node:**  
   - Type: `set`  
   - Add two string assignments:  
     - Name: "Person_info" with expression `={{ $json["Person Info"] || $json.Person_info }}`  
     - Name: "Airtop_profile" with expression `={{ $json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile }}`  
   - This node normalizes inputs from form or workflow.

4. **Connect:**  
   - Connect "On form submission" → "Unify Parameters"  
   - Connect "When Executed by Another Workflow" → "Unify Parameters"

5. **Create "Search Profile URL" node:**  
   - Type: `airtop`  
   - Credentials: Airtop API Key with organization (e.g., "Airtop Official Org")  
   - Parameters:  
     - URL: Use expression `=https://www.google.com/search?q={{ encodeURI($json['Person_info']+' Linkedin') }}`  
     - Prompt: Use the text instructing extraction of the most likely LinkedIn URL or return "NA"  
     - Resource: "extraction"  
     - Operation: "query"  
     - Session mode: "new"  
   - Connect "Unify Parameters" → "Search Profile URL"

6. **Create "Is valid Linkedin link?" node:**  
   - Type: `filter`  
   - Conditions (all must be true):  
     - `Airtop_profile` is not empty: expression `={{ $('Unify Parameters').item.json.Airtop_profile }}` not empty  
     - Extracted LinkedIn URL does not start with "NA" (expression on `$json.data.modelResponse`)  
     - Extracted LinkedIn URL contains "linkedin.com/in"  
   - Connect "Search Profile URL" → "Is valid Linkedin link?"

7. **Create "Verify Profile URL" node:**  
   - Type: `airtop`  
   - Credentials: Airtop API Key with organization  
   - Parameters:  
     - URL: expression `={{ $('Search Profile URL').item.json.data.modelResponse }}`  
     - Prompt: Instruction to analyze experience and confirm profile matches "Person_info"  
     - Profile Name: expression `={{ $('Unify Parameters').item.json.Airtop_profile }}` (authenticated Airtop profile)  
     - Resource: "extraction"  
     - Operation: "query"  
     - Session mode: "new"  
   - Connect "Is valid Linkedin link?" → "Verify Profile URL"

8. **Optionally add Sticky Notes** for documentation:  
   - Input parameters note near triggers  
   - Search and verification explanation near search and filter nodes  
   - Full README overview as a large sticky note at the top-left corner

9. **Activate the workflow** and test by submitting the form or triggering via another workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This automation locates and verifies a LinkedIn profile using Google search plus an authenticated Airtop browser profile for verification. It is ideal for marketing, recruiting, or contact enrichment workflows.                                                                                                                                                                                      | General workflow purpose                                |
| Airtop API Key and optionally an Airtop LinkedIn authenticated browser profile are required to use this workflow effectively.                                                                                                                                                                                                                                                                         | https://portal.airtop.ai/api-keys                       |
| Airtop Profiles can be created and managed at https://portal.airtop.ai/browser-profiles                                                                                                                                                                                                                                                                                                              | https://portal.airtop.ai/browser-profiles               |
| The workflow can be triggered via a web form or invoked by another workflow to enable flexible integration in automation pipelines.                                                                                                                                                                                                                                                                  | Workflow trigger options                                |
| Potential enhancements include combining this workflow with email-to-profile lookup tools or CRM integration for automated contact enrichment and outreach.                                                                                                                                                                                                                                         | Suggested next steps                                    |
| The workflow may take several minutes in the Airtop nodes due to the complexity of Google search scraping and profile content analysis. Please account for this latency when integrating.                                                                                                                                                                                                             | Performance considerations                              |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.