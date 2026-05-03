Find Linkedin Company Page with Airtop

https://n8nworkflows.xyz/workflows/find-linkedin-company-page-with-airtop-4254


# Find Linkedin Company Page with Airtop

### 1. Workflow Overview

This workflow automates the discovery of a company's official LinkedIn page by leveraging a combination of direct website scraping, LinkedIn search, and Google search using the Airtop API. It is designed for users who have authenticated Airtop Profiles connected to LinkedIn and want to streamline outreach, research, or enrichment tasks by reliably finding the LinkedIn company page from a given company domain.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Accepts input parameters either via a form submission or when triggered by another workflow.
- **1.2 Parameter Unification**: Normalizes and prepares input parameters for consistent usage downstream.
- **1.3 Company Website Scraping**: Uses Airtop to extract a LinkedIn URL directly from the company's web domain.
- **1.4 LinkedIn Search**: If no valid LinkedIn link is found on the website, performs a LinkedIn search using Airtop.
- **1.5 Google Search Fallback**: If LinkedIn search fails, performs a Google search to find the LinkedIn page.
- **1.6 LinkedIn URL Verification**: Validates the extracted LinkedIn URL by invoking a dedicated verification sub-workflow.
- **1.7 Output Mapping**: Prepares the final output with the verified LinkedIn URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block captures the workflow input parameters either from a web form or from another workflow.
- **Nodes Involved:** 
  - On form submission
  - When Executed by Another Workflow
  - Sticky Note1

- **Node Details:**

  - **On form submission**
    - Type: Form Trigger  
    - Role: Entry point accepting user input via a web form.
    - Configuration:  
      - Form titled "Company LinkedIn" with two required fields:  
        - "Company domain" (e.g., company.com)  
        - "Airtop Profile (connected to Linkedin)" (name of the authenticated Airtop profile)  
      - Form description explains the automation and provides a link to create Airtop profiles.
    - Inputs: External user submission  
    - Outputs: JSON object with form field data  
    - Edge cases: Missing required fields; malformed domain input.
  
  - **When Executed by Another Workflow**
    - Type: Execute Workflow Trigger  
    - Role: Allows this workflow to be invoked programmatically by other workflows, passing the same input parameters.  
    - Configuration: Defines expected inputs "Company domain" and "Airtop Profile (connected to Linkedin)".  
    - Inputs: Input from another workflow  
    - Outputs: Passes input data downstream  
    - Edge cases: Missing or invalid input parameters from upstream workflows.

  - **Sticky Note1**
    - Role: Documentation note indicating this block handles input parameters via form or workflow call.

---

#### 2.2 Parameter Unification

- **Overview:** Normalizes and stores input parameters into a consistent JSON structure for downstream use.
- **Nodes Involved:** 
  - Unify parameters
- **Node Details:**

  - **Unify parameters**
    - Type: Set  
    - Role: Extracts and renames input fields to standardized parameter names.  
    - Configuration: Assigns:  
      - "Company domain" from either form or workflow input  
      - "Airtop Profile" from input "Airtop Profile (connected to Linkedin)"  
    - Inputs: Raw input JSON from trigger nodes  
    - Outputs: JSON with unified keys for "Company domain" and "Airtop Profile"  
    - Edge cases: Missing or malformed inputs; no validation here.

---

#### 2.3 Company Website Scraping

- **Overview:** Attempts to extract the company's LinkedIn page URL by querying the company's website using an Airtop profile session.
- **Nodes Involved:** 
  - Webpage search (Airtop node)  
  - LinkedIn link found? (If node)  
  - Sticky Note (Query webpage)  

- **Node Details:**

  - **Webpage search**
    - Type: Airtop node  
    - Role: Queries the company domain URL, instructing Airtop to locate a LinkedIn company page link, typically found in the footer.  
    - Configuration:  
      - URL parameter dynamically set to the company domain  
      - Prompt instructs to return only the LinkedIn link found  
      - Uses the Airtop profile passed from unified parameters  
      - New session mode to avoid stale data  
      - Airtop API credentials configured  
    - Inputs: Unified parameters  
    - Outputs: JSON containing extracted modelResponse with the link or empty  
    - Edge cases: Website unreachable, no LinkedIn link found, rate limits or Airtop API errors.

  - **LinkedIn link found?** (first instance)
    - Type: If  
    - Role: Checks if the extracted link contains "linkedin.com/company" indicating a valid LinkedIn company page link.  
    - Configuration: String contains condition on modelResponse  
    - Inputs: Output of Webpage search  
    - Outputs:  
      - True: Proceed to verification  
      - False: Continue to LinkedIn search block  
    - Edge cases: modelResponse missing or malformed.

  - **Sticky Note**  
    - Content explains this block attempts to find LinkedIn profile link by querying the company webpage.

---

#### 2.4 LinkedIn Search

- **Overview:** If no LinkedIn URL is found on the company website, this block performs a LinkedIn company search using Airtop.
- **Nodes Involved:**  
  - Linkedin search (Airtop node)  
  - LinkedIn link found?1 (If node)  
  - Sticky Note2 (Search LinkedIn)  

- **Node Details:**

  - **Linkedin search**
    - Type: Airtop node  
    - Role: Searches LinkedIn for company pages using a crafted URL with keywords derived from the company domain.  
    - Configuration:  
      - URL: LinkedIn search URL with encoded keywords derived from domain parsing logic (handles country code TLDs)  
      - Prompt instructs to return only the most likely LinkedIn URL  
      - Uses Airtop profile from unified parameters  
      - New session mode  
      - Max retries: 2  
      - Airtop credentials  
    - Inputs: Unified parameters, downstream of failed website search  
    - Outputs: JSON modelResponse with LinkedIn URL or empty  
    - Edge cases: LinkedIn blocking, rate limits, empty/incorrect results.

  - **LinkedIn link found?1**
    - Type: If  
    - Role: Checks if LinkedIn search result contains a valid LinkedIn company page URL.  
    - Configuration: Same string contains check as previous if node.  
    - Inputs: Linkedin search output  
    - Outputs:  
      - True: Proceed to company LinkedIn exists? block (verification)  
      - False: Proceed to Google search fallback  
    - Edge cases: No results or malformed data.

  - **Sticky Note2**  
    - Explains this block runs LinkedIn search if webpage scraping fails or finds no LinkedIn link.

---

#### 2.5 Google Search Fallback

- **Overview:** If LinkedIn search fails, performs a Google search to find the LinkedIn company page as a last resort.
- **Nodes Involved:**  
  - Google search (Airtop node)  
  - Company LinkedIn exists? (Filter node)  
  - Sticky Note2 (Search LinkedIn, also covers this fallback)  

- **Node Details:**

  - **Google search**
    - Type: Airtop node  
    - Role: Searches Google for the company LinkedIn page by querying "company domain LinkedIn".  
    - Configuration:  
      - URL: Google search URL with encoded query of company domain + "LinkedIn"  
      - Prompt instructs to return only the most likely LinkedIn URL or "NA" if not found  
      - Session mode new  
      - Airtop credentials  
      - Max retries: 2  
    - Inputs: From LinkedIn link found?1 false branch  
    - Outputs: JSON modelResponse with LinkedIn URL or "NA"  
    - Edge cases: Google blocking, no results, "NA" result.

  - **Company LinkedIn exists?**
    - Type: Filter  
    - Role: Filters results to check if the modelResponse contains a valid LinkedIn company URL ("linkedin.com/company").  
    - Inputs: Google search output  
    - Outputs:  
      - True: Passes to verification  
      - False: Ends or could be extended for further handling  
    - Edge cases: No or invalid URL found.

---

#### 2.6 LinkedIn URL Verification

- **Overview:** Verifies the authenticity and correctness of the LinkedIn URL by invoking a separate verification workflow.
- **Nodes Involved:**  
  - Verify LinkedIn link (Execute Workflow)  
  - Map data (Set)  
  - Sticky Note3 (Verify LinkedIn URL)  

- **Node Details:**

  - **Verify LinkedIn link**
    - Type: Execute Workflow  
    - Role: Calls sub-workflow "AIRTOP — Verify Company LinkedIn by Domain" to validate the LinkedIn URL against the company domain.  
    - Configuration:  
      - Passes parameters:  
        - Company Domain (from unified parameters)  
        - Company LinkedIn (modelResponse from previous step)  
        - Airtop Profile (unified parameter)  
      - Input/output schema defined for matching and conversion  
    - Inputs: LinkedIn URL candidates from previous blocks  
    - Outputs: Validated LinkedIn URL or rejection  
    - Edge cases: Sub-workflow errors, invalid URL, network/API issues.

  - **Map data**
    - Type: Set  
    - Role: Prepares final output by setting the field "company_linkedin" to the verified LinkedIn URL.  
    - Inputs: Output of Verify LinkedIn link node  
    - Outputs: Final output JSON for downstream consumption or return  
    - Edge cases: Missing or invalid verification result.

  - **Sticky Note3**  
    - Notes that this block performs verification of the LinkedIn URL.

---

#### 2.7 Documentation and Notes

- **Nodes Involved:**  
  - Sticky Note7  

- **Node Details:**

  - **Sticky Note7**
    - Content: Extensive README style documentation embedded directly in the workflow.  
    - Describes use case, workflow steps, setup requirements, and next steps for integration.  
    - Provides helpful URLs for Airtop profile and API key setup.  
    - Serves as a single source of truth for users and maintainers.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                        | Input Node(s)                | Output Node(s)                 | Sticky Note                                                      |
|-----------------------------|-------------------------|-------------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------------------|
| On form submission          | Form Trigger            | Entry point via form input           | -                           | Unify parameters              | ## Input Parameters Run this workflow using a form or from another workflow |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point via workflow call        | -                           | Unify parameters              | ## Input Parameters Run this workflow using a form or from another workflow |
| Unify parameters            | Set                     | Normalize input parameters           | On form submission, When Executed by Another Workflow | Webpage search               | ## Query webpage Search the provided company webpage for a LinkedIn profile link. |
| Webpage search              | Airtop                  | Extract LinkedIn URL from company website | Unify parameters            | LinkedIn link found?          | ## Query webpage Search the provided company webpage for a LinkedIn profile link. |
| LinkedIn link found?        | If                      | Check if website scraping found LinkedIn URL | Webpage search              | Verify LinkedIn link, Linkedin search | ## Search LinkedIn If the webpage can't be found or doesn't contain a LinkedIn link, a search is performed on LinkedIn (and then on Google) |
| Linkedin search             | Airtop                  | Search LinkedIn company pages       | LinkedIn link found? (false branch) | LinkedIn link found?1         | ## Search LinkedIn If the webpage can't be found or doesn't contain a LinkedIn link, a search is performed on LinkedIn (and then on Google) |
| LinkedIn link found?1       | If                      | Check if LinkedIn search found URL  | Linkedin search             | Company LinkedIn exists?, Google search | ## Search LinkedIn If the webpage can't be found or doesn't contain a LinkedIn link, a search is performed on LinkedIn (and then on Google) |
| Google search              | Airtop                  | Fallback Google search for LinkedIn | LinkedIn link found?1 (false branch) | Company LinkedIn exists?      | ## Search LinkedIn If the webpage can't be found or doesn't contain a LinkedIn link, a search is performed on LinkedIn (and then on Google) |
| Company LinkedIn exists?    | Filter                  | Check if Google search found LinkedIn URL | Google search              | Verify LinkedIn link          | ## Verify LinkedIn URL                                           |
| Verify LinkedIn link        | Execute Workflow        | Validate LinkedIn URL correctness   | LinkedIn link found?, Company LinkedIn exists? | Map data                     | ## Verify LinkedIn URL                                           |
| Map data                   | Set                     | Prepare final output with verified URL | Verify LinkedIn link       | -                             | ## Verify LinkedIn URL                                           |
| Sticky Note1               | Sticky Note             | Documentation for input parameters  | -                           | -                             | ## Input Parameters Run this workflow using a form or from another workflow |
| Sticky Note                | Sticky Note             | Documentation for webpage query     | -                           | -                             | ## Query webpage Search the provided company webpage for a LinkedIn profile link. |
| Sticky Note2               | Sticky Note             | Documentation for LinkedIn/Google search fallback | -                           | -                             | ## Search LinkedIn If the webpage can't be found or doesn't contain a LinkedIn link, a search is performed on LinkedIn (and then on Google) |
| Sticky Note3               | Sticky Note             | Documentation for LinkedIn URL verification | -                           | -                             | ## Verify LinkedIn URL                                           |
| Sticky Note7               | Sticky Note             | README and detailed workflow overview | -                           | -                             | README: Automating LinkedIn Company Page Discovery with setup and use case details |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission")**
   - Type: Form Trigger  
   - Configure form with title "Company LinkedIn" and two required fields:  
     - "Company domain" (placeholder: company.com)  
     - "Airtop Profile (connected to Linkedin)"  
   - Add form description with HTML content explaining the automation and link to Airtop profiles.

2. **Create an Execute Workflow Trigger Node ("When Executed by Another Workflow")**
   - Accepts inputs: "Company domain" and "Airtop Profile (connected to Linkedin)".

3. **Create a Set Node ("Unify parameters")**
   - Assign two string fields from input data:  
     - "Company domain" from input "Company domain"  
     - "Airtop Profile" from input "Airtop Profile (connected to Linkedin)"  
   - Connect outputs of "On form submission" and "When Executed by Another Workflow" to this node.

4. **Create an Airtop Node ("Webpage search")**
   - Resource: extraction  
   - Operation: query  
   - URL: `={{ $json["Company domain"] }}`  
   - Prompt: "This is the webpage for a company. Search for the linkedin of the company, it is highly probable that it is in the footer. Return only the link"  
   - Profile Name: `={{ $('Unify parameters').item.json["Airtop Profile"] }}`  
   - Session Mode: new  
   - Set Airtop API credentials (use your Airtop API key).  
   - Connect output of "Unify parameters" to this node.

5. **Create an If Node ("LinkedIn link found?")**
   - Condition: Check if `data.modelResponse` contains "linkedin.com/company" (case sensitive, strict).  
   - Connect output of "Webpage search" to this node.

6. **On True Branch of "LinkedIn link found?"**
   - Connect to a new Execute Workflow Node ("Verify LinkedIn link") to validate the link.

7. **On False Branch of "LinkedIn link found?"**
   - Create an Airtop Node ("Linkedin search")  
     - URL: LinkedIn company search URL constructed as:  
       ```
       https://www.linkedin.com/search/results/companies/?keywords={{ encodeURIComponent(
         ['org','com','co','fr','us','uk','de','es','it','nl','ca','au','in','jp'].includes($('Unify parameters').item.json["Company domain"].split('.').at(-1))
         ? $('Unify parameters').item.json["Company domain"].split('.')[0] 
         : $('Unify parameters').item.json["Company domain"].split('.').join(" ")
       ) }}
       ```  
     - Prompt: "This is Linked Search results. One of the first results should be the Linkedin Page of {{ company domain }}. Return the Linkedin URL of the most likely page and nothing else."  
     - Profile Name: `={{ $('Unify parameters').item.json["Airtop Profile"] }}`  
     - Session Mode: new  
     - Max retries: 2  
     - Airtop API credentials  
   - Connect false branch of "LinkedIn link found?" to this node.

8. **Create an If Node ("LinkedIn link found?1")**
   - Condition: Check if `data.modelResponse` contains "linkedin.com/company".  
   - Connect output of "Linkedin search" to this node.

9. **On True Branch of "LinkedIn link found?1"**
   - Connect to a Filter Node ("Company LinkedIn exists?") to verify the existence of the company LinkedIn.

10. **On False Branch of "LinkedIn link found?1"**
    - Create Airtop Node ("Google search")  
      - URL:  
        ```
        https://www.google.com/search?q={{ encodeURIComponent(`${$('Unify parameters').item.json["Company domain"]} LinkedIn`) }}
        ```  
      - Prompt:  
        ```
        This is Google Search results. One of the first results should be the Linkedin Page of {{ company domain }}.
        Return the Linkedin URL of the most likely page and nothing else.
        If can't find the URL return 'NA'
        A valid Linkedin profile URL starts with "https://www.linkedin.com/company/"
        ```  
      - Profile Name: empty string `=""` (use default or specify profile if needed)  
      - Session Mode: new  
      - Max retries: 2  
      - Airtop API credentials  
    - Connect false branch of "LinkedIn link found?1" to this node.

11. **Create a Filter Node ("Company LinkedIn exists?")**
    - Condition: Check if `data.modelResponse` contains "linkedin.com/company".  
    - Connect output of "Google search" to this node.

12. **Connect True Branch of "Company LinkedIn exists?"**
    - To "Verify LinkedIn link" Execute Workflow node.

13. **Create an Execute Workflow Node ("Verify LinkedIn link")**
    - Workflow ID: Reference the existing workflow "AIRTOP — Verify Company LinkedIn by Domain" or create equivalent.  
    - Workflow inputs:  
      - Company Domain: `={{ $('Unify parameters').item.json["Company domain"] }}`  
      - Company LinkedIn: `={{ $json.data.modelResponse }}`  
      - Airtop Profile (connected to LinkedIn): `={{ $('Unify parameters').item.json["Airtop Profile"] }}`  
    - Connect outputs of all verification entry points to this node.

14. **Create a Set Node ("Map data")**
    - Assign field "company_linkedin" with value `={{ $json.company_linkedin }}` from verification output.  
    - Connect output of "Verify LinkedIn link" to this node.

15. **Add Sticky Notes**
    - Add comprehensive sticky notes to document input parameters, query webpage, search fallbacks, verification, and overall README as per original workflow.

16. **Configure Credentials**
    - Import or create Airtop API credentials with valid API key.  
    - Assign credentials properly to all Airtop nodes.

17. **Activate and Test**
    - Test the workflow by submitting form or triggering via another workflow with sample inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This automation requires an Airtop Profile authenticated on LinkedIn. Users can create a free profile and log in at https://portal.airtop.ai/browser-profiles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Airtop Profile Setup                                                                                        |
| Setup requires an Airtop API Key, which can be generated for free at https://portal.airtop.ai/api-keys.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Airtop API Key Setup                                                                                        |
| The workflow uses a multi-step fallback approach: Website scraping → LinkedIn search → Google search, to maximize the chance of finding the correct LinkedIn company page.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow logic description                                                                                  |
| The verification sub-workflow "AIRTOP — Verify Company LinkedIn by Domain" is critical for ensuring data quality by confirming the LinkedIn URL corresponds to the domain.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sub-workflow dependency                                                                                      |
| This workflow can be combined with People Enrichment workflows or CRM integrations to provide enriched company data automatically.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Suggested next steps for integration                                                                         |
| For more details about Airtop Profiles and usage, visit https://portal.airtop.ai/browser-profiles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Airtop Profiles documentation                                                                                |
| Embedded README sticky note inside the workflow provides detailed operational and setup instructions for users and maintainers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Documentation included in Sticky Note7                                                                       |

---

**Disclaimer:**  
The provided description and documentation are based exclusively on an automated workflow created with n8n, adhering strictly to applicable content policies. All processed data is legal and public.