Sales Prospect Research & Outreach Preparation with Apollo, Linkup AI, and LinkedIn

https://n8nworkflows.xyz/workflows/sales-prospect-research---outreach-preparation-with-apollo--linkup-ai--and-linkedin-6492


# Sales Prospect Research & Outreach Preparation with Apollo, Linkup AI, and LinkedIn

### 1. Workflow Overview

This workflow, titled **"Sales Prospect Research & Outreach Preparation with Apollo, Linkup AI, and LinkedIn"**, automates the process of enriching a sales prospect’s profile and generating tailored outreach insights. It is designed for sales and marketing professionals seeking to personalize their outreach by leveraging AI-powered research on a prospect’s LinkedIn profile combined with business context.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures prospect data (First name, Last name, Company) via a web form.
- **1.2 Prospect Enrichment with Apollo:** Uses the Apollo API to enrich the prospect’s data, retrieving professional details including the LinkedIn profile URL.
- **1.3 Business Context Definition:** Injects user-defined business context and value proposition into the workflow to guide AI processing.
- **1.4 LinkedIn Profile Analysis with Linkup AI:** Sends a prompt to Linkup AI to analyze the LinkedIn profile and generate structured summaries about the prospect’s background, pain points, and how the user’s offering could help.
- **1.5 Results Consolidation:** Aggregates data from all previous steps into a structured output for downstream use or export.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  Captures user-submitted prospect information via an embedded form, triggering the workflow.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **Node Name:** On form submission  
    - **Type:** Form Trigger  
    - **Technical Role:** Entry point, listens for form submissions with prospect data  
    - **Configuration:**  
      - Webhook path: `find-enrich-contact`  
      - Form title: "Contact to enrich"  
      - Fields: First name (required), Last name (required), Company name (required)  
    - **Expressions/Variables:** Form fields map directly to JSON keys: `First name`, `Last name`, `Company name`  
    - **Connections:** Output connected to "Enrich contact with Apollo" node  
    - **Failure/Edge Cases:** Form validation errors if required fields are missing; webhook availability issues  
    - **Version Requirements:** Supports n8n version with formTrigger node v2.2+  
    - **Sub-workflow:** None

---

#### 2.2 Prospect Enrichment with Apollo

- **Overview:**  
  Queries Apollo API to match the prospect by name and company, retrieving detailed contact and company information, including LinkedIn URL.

- **Nodes Involved:**  
  - Enrich contact with Apollo

- **Node Details:**

  - **Node Name:** Enrich contact with Apollo  
    - **Type:** HTTP Request  
    - **Technical Role:** External API call to Apollo to enrich contact data  
    - **Configuration:**  
      - Method: POST  
      - URL constructed dynamically using incoming form data, URL-encoded:  
        `https://api.apollo.io/api/v1/people/match?first_name={{First name}}&last_name={{Last name}}&organization_name={{Company name}}&reveal_personal_emails=false&reveal_phone_number=false`  
      - Headers include `Cache-Control: no-cache` and `Accept: application/json`  
      - Authentication: HTTP Header Auth with Apollo API key credential  
    - **Expressions/Variables:** Uses expressions to URL encode form inputs  
    - **Connections:** Output connected to "Define our business context" node  
    - **Failure/Edge Cases:**  
      - HTTP errors (4xx, 5xx) if API key invalid or rate limited  
      - Empty or ambiguous match results from Apollo  
      - Network timeouts  
    - **Version Requirements:** HTTP Request node v4.2+ recommended  
    - **Sub-workflow:** None

---

#### 2.3 Business Context Definition

- **Overview:**  
  Sets static business-specific information that contextualizes the AI analysis of the prospect profile.

- **Nodes Involved:**  
  - Define our business context

- **Node Details:**

  - **Node Name:** Define our business context  
    - **Type:** Set Node  
    - **Technical Role:** Assigns static strings describing the area of pain points and the user’s offering  
    - **Configuration:**  
      - Field "Area for which the prospect could experience pain points" with a descriptive string about selling coding bootcamps as a solopreneur  
      - Field "My offering" with a detailed description of online marketing services and bootcamp platform deals  
    - **Expressions/Variables:** Static strings, no expressions used here  
    - **Connections:** Output connected to "Find Linkedin profile information with Linkup" node  
    - **Failure/Edge Cases:** None significant, but missing or inaccurate business context reduces AI output quality  
    - **Version Requirements:** Set node v3.4+  
    - **Sub-workflow:** None

---

#### 2.4 LinkedIn Profile Analysis with Linkup AI

- **Overview:**  
  Sends a structured prompt to the Linkup AI API, asking for a qualitative summary and pain point analysis based on the prospect’s LinkedIn URL enriched from Apollo.

- **Nodes Involved:**  
  - Find Linkedin profile information with Linkup

- **Node Details:**

  - **Node Name:** Find Linkedin profile information with Linkup  
    - **Type:** HTTP Request  
    - **Technical Role:** Calls Linkup AI API to analyze LinkedIn profile and generate structured JSON output with summaries  
    - **Configuration:**  
      - Method: POST  
      - URL: `https://api.linkup.so/v1/search`  
      - Body parameters include:  
        - `q`: prompt string embedding LinkedIn URL from Apollo output  
        - `depth`: "standard"  
        - `outputType`: "structured"  
        - `structuredOutputSchema`: JSON schema defining required fields:  
          - `about_that_person_summary` (string)  
          - `potential_pain_points_summary` (string)  
          - `how_we_could_bring_them_value_summary` (string)  
        - `includeImages`: false  
      - Authentication: HTTP Bearer Auth with Linkup API key credential  
    - **Expressions/Variables:** Uses expressions to inject LinkedIn URL and business context fields into the prompt and schema  
    - **Connections:** Output connected to "Consolidate results" node  
    - **Failure/Edge Cases:**  
      - API rate limits or invalid tokens  
      - Malformed or missing LinkedIn URLs from Apollo enrichment  
      - Parsing errors if API response doesn’t match schema  
      - Timeout or connectivity issues  
    - **Version Requirements:** HTTP Request v4.2+  
    - **Sub-workflow:** None

---

#### 2.5 Results Consolidation

- **Overview:**  
  Combines all collected and generated data into a unified JSON object, suitable for export or further processing.

- **Nodes Involved:**  
  - Consolidate results

- **Node Details:**

  - **Node Name:** Consolidate results  
    - **Type:** Set Node  
    - **Technical Role:** Aggregates fields from previous nodes into a final output format  
    - **Configuration:**  
      - Maps individual fields such as First name, Last name, Company name from form submission  
      - Maps LinkedIn URL, Job title, Email, Company info, and AI-generated summaries from Apollo and Linkup responses  
    - **Expressions/Variables:** Uses expressions referencing previous nodes’ outputs, e.g. `$('On form submission').item.json['First name']`, `$json.about_that_person_summary`  
    - **Connections:** Terminal node (no output connections)  
    - **Failure/Edge Cases:** Missing data in any referenced node will propagate as empty or null fields  
    - **Version Requirements:** Set node v3.4+  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                      | Input Node(s)           | Output Node(s)                      | Sticky Note                                                                                  |
|-------------------------------|--------------------|------------------------------------|-------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger       | Capture prospect data via form     |                         | Enrich contact with Apollo        |                                                                                              |
| Enrich contact with Apollo     | HTTP Request       | Enrich contact data via Apollo API | On form submission       | Define our business context       | Connect your Apollo credential. Get an API key at: apollo.io                                 |
| Define our business context    | Set                | Inject business context strings    | Enrich contact with Apollo | Find Linkedin profile information with Linkup | Document your business context. The more details the better                                   |
| Find Linkedin profile information with Linkup | HTTP Request | Analyze LinkedIn profile with Linkup AI | Define our business context | Consolidate results               | Connect your Linkup credential. Get a free API key at: https://www.linkup.so/                |
| Consolidate results            | Set                | Aggregate all data into final output | Find Linkedin profile information with Linkup |                                   |                                                                                              |
| Sticky Note1                  | Sticky Note        | Workflow description and instructions |                         |                                   | AI Sales Research Assistant overview and usage instructions                                  |
| Sticky Note                   | Sticky Note        | Apollo credential reminder         |                         |                                   | Connect your Apollo credential. Get an API key at: apollo.io                                 |
| Sticky Note3                  | Sticky Note        | Business context reminder          |                         |                                   | Document your business context. The more details the better                                  |
| Sticky Note2                  | Sticky Note        | Linkup credential reminder         |                         |                                   | Connect your Linkup credential. Get a free API key at: https://www.linkup.so/                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**  
   - Name: `On form submission`  
   - Configure webhook path: `find-enrich-contact`  
   - Form Title: `Contact to enrich`  
   - Add three required fields:  
     - First name (placeholder: Jane)  
     - Last name (placeholder: Doe)  
     - Company name (placeholder: Acme Corp)  
   - This node will trigger the workflow on form submission.

3. **Add an HTTP Request node:**  
   - Name: `Enrich contact with Apollo`  
   - Method: POST  
   - URL:  
     ``` 
     https://api.apollo.io/api/v1/people/match?first_name={{ $json["First name"].urlEncode() }}&last_name={{ $json["Last name"].urlEncode() }}&organization_name={{ $json["Company name"].urlEncode() }}&reveal_personal_emails=false&reveal_phone_number=false
     ```  
   - Headers:  
     - Cache-Control: no-cache  
     - Accept: application/json  
   - Authentication: Set to HTTP Header Auth using your Apollo API key credential  
   - Connect output of `On form submission` to this node.

4. **Add a Set node:**  
   - Name: `Define our business context`  
   - Add two string fields:  
     - `Area for which the prospect could experience pain points` with your description (e.g., "Getting online visibility to sell coding bootcamps as a solopreneur")  
     - `My offering` with your product/service description (e.g., online marketing services, SEO, social media campaigns, etc.)  
   - Connect output of `Enrich contact with Apollo` to this node.

5. **Add another HTTP Request node:**  
   - Name: `Find Linkedin profile information with Linkup`  
   - Method: POST  
   - URL: `https://api.linkup.so/v1/search`  
   - Authentication: HTTP Bearer Auth using your Linkup API key credential  
   - Body parameters (JSON / form-data):  
     - `q` (string):  
       ```
       Research information on this linkedin profile:
       {{ $('Enrich contact with Apollo').item.json.person.linkedin_url }}
       ```  
     - `depth`: "standard"  
     - `outputType`: "structured"  
     - `structuredOutputSchema` (JSON string):  
       Define a schema with required string fields: `about_that_person_summary`, `potential_pain_points_summary`, `how_we_could_bring_them_value_summary`  
       Include descriptions referencing business context fields using expressions.  
     - `includeImages`: false  
   - Connect output of `Define our business context` to this node.

6. **Add a final Set node:**  
   - Name: `Consolidate results`  
   - Map fields from all previous nodes with expressions:  
     - First name, Last name, Company name from form submission node  
     - LinkedIn URL, Job title, Email, Company website, LinkedIn URL, Industry, Employee count, Description from Apollo node  
     - AI-generated summaries from Linkup node: about_that_person_summary, potential_pain_points_summary, how_we_could_bring_them_value_summary  
   - Connect output of `Find Linkedin profile information with Linkup` to this node.

7. **Add Sticky Notes (optional but recommended):**  
   - Add notes reminding users to:  
     - Connect Apollo API credential (with link to apollo.io)  
     - Connect Linkup API credential (with link https://www.linkup.so/)  
     - Document business context well  
     - Provide an overview of workflow purpose and usage instructions

8. **Set credentials for HTTP nodes:**  
   - Create and configure HTTP Header Auth credential for Apollo API key.  
   - Create and configure HTTP Bearer Auth credential for Linkup API key.

9. **Activate the workflow and test by submitting the form at the webhook URL.**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                      |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow finds a person, analyzes their profile, and generates AI-based summaries for sales outreach.| Overview and usage instructions in Sticky Note1     |
| Apollo API key is required for data enrichment.                                                      | https://apollo.io                                    |
| Linkup API key is required for AI-powered LinkedIn analysis.                                         | https://www.linkup.so/                               |
| Business context definitions critically influence AI summary quality.                                | Define our business context node                      |

---

**Disclaimer:**  
The provided text is generated from an n8n workflow automation. It complies strictly with applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.