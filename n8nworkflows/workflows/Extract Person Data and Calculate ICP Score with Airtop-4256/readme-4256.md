Extract Person Data and Calculate ICP Score with Airtop

https://n8nworkflows.xyz/workflows/extract-person-data-and-calculate-icp-score-with-airtop-4256


# Extract Person Data and Calculate ICP Score with Airtop

### 1. Workflow Overview

This workflow automates the enrichment of a person’s professional data and calculates an Ideal Customer Profile (ICP) score by leveraging LinkedIn information accessed through Airtop browser profiles. It is designed for lead qualification, contact research, and sales or marketing outreach optimization.

The workflow accepts inputs either directly from a web form submission or triggered by another workflow, then validates and standardizes the inputs, verifies the corporate nature of the email, discovers the LinkedIn profile URL, extracts detailed LinkedIn data, calculates the ICP score based on this data, and finally merges all enriched information into a consolidated output.

**Logical Blocks:**

- **1.1 Input Reception and Normalization**  
  Captures input parameters via form or workflow trigger, then unifies parameter naming for consistent downstream usage.

- **1.2 Corporate Email Validation**  
  Filters out non-corporate/free email addresses to ensure quality of data enrichment.

- **1.3 LinkedIn Profile Discovery and Verification**  
  Uses Airtop to find and verify the LinkedIn URL associated with the person.

- **1.4 Data Enrichment and ICP Scoring**  
  Extracts detailed person data from LinkedIn and calculates an ICP score via separate sub-workflows.

- **1.5 Output Consolidation**  
  Merges enriched data and scoring results into a single output for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Normalization

- **Overview:**  
  This block gathers input parameters from two possible sources: a web form submission or an external workflow trigger. It then standardizes these inputs into unified parameter names to simplify processing.

- **Nodes Involved:**  
  - On form submission  
  - When Executed by Another Workflow  
  - Unify Parameters  
  - Sticky Note (Input Parameters)

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Receives user input from a web form titled "Linkedin Profile Extractor" with required fields: Person Name, Work Email, Airtop Profile.  
    - *Configuration:* Form description explains the purpose and links to Airtop profile creation.  
    - *Input/Output:* Incoming HTTP webhook request; output to "Unify Parameters".  
    - *Potential Failures:* Webhook connectivity issues, missing required fields.  

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Allows this workflow to be triggered programmatically by another workflow, accepting the same 3 parameters.  
    - *Input/Output:* Inputs: person_name, work_email, Airtop_profile; output to "Unify Parameters".  
    - *Potential Failures:* Missing expected inputs, triggering errors.

  - **Unify Parameters**  
    - *Type:* Set  
    - *Role:* Consolidates input keys from both entry points into uniform variable names: Person_name, Work_email, Airtop_profile.  
    - *Key Expressions:* Uses expressions like `{{$json["Person Name"] || $json.person_name}}` to take input from either form or workflow trigger.  
    - *Input/Output:* Inputs from either form or workflow trigger; outputs unified JSON to "Is corporate email?".  
    - *Potential Failures:* Expression evaluation errors if input keys missing unexpectedly.

  - **Sticky Note**  
    - *Content:* "## Input Parameters\n** Run this workflow using a form or from another workflow"  
    - *Purpose:* Documentation for users on input options.

---

#### 2.2 Corporate Email Validation

- **Overview:**  
  Filters out email addresses that are not corporate (i.e., excludes common free or personal email domains) to ensure subsequent enrichment steps operate on business contacts.

- **Nodes Involved:**  
  - Is corporate email?

- **Node Details:**

  - **Is corporate email?**  
    - *Type:* Filter  
    - *Role:* Evaluates if the Work_email field does NOT end with or contain free email domains such as @gmail.com, @hotmail., .edu, .net, and others listed.  
    - *Configuration:* Multiple conditions combined with AND, all negating free email indicators.  
    - *Input/Output:* Input from "Unify Parameters"; on pass outputs to "Find Person Linkedin URL", else stops.  
    - *Potential Failures:* Edge case emails not covered by rules might pass incorrectly; false negatives if corporate email resembles free domains.

---

#### 2.3 LinkedIn Profile Discovery and Verification

- **Overview:**  
  Uses the Airtop service to find a LinkedIn URL for the person based on their name and email, then verifies the validity of the resulting URL.

- **Nodes Involved:**  
  - Find Person Linkedin URL  
  - Is valid Linkedin URL?

- **Node Details:**

  - **Find Person Linkedin URL**  
    - *Type:* Execute Workflow  
    - *Role:* Calls a sub-workflow ("AIRTOP — LinkedIn Profile Discovery w Verification") that queries Airtop to discover and verify the LinkedIn profile URL using Person_name and Work_email along with Airtop_profile credentials.  
    - *Input/Output:* Input parameters: Person_info (combined name and email), Airtop_profile; output LinkedIn URL in `data.modelResponse` to "Is valid Linkedin URL?".  
    - *Potential Failures:* Airtop API errors, profile not found, network timeouts.

  - **Is valid Linkedin URL?**  
    - *Type:* Filter  
    - *Role:* Checks that the LinkedIn URL is not empty and does not start with "NA" (indicating failure or no result).  
    - *Input/Output:* Input from "Find Person Linkedin URL"; if valid, routes to "Extract Person Data" and "ICP Scoring"; else stops.  
    - *Potential Failures:* Invalid URL formats, false positives if API returns unexpected values.

---

#### 2.4 Data Enrichment and ICP Scoring

- **Overview:**  
  Enriches the contact’s profile by extracting detailed data from LinkedIn and calculates an ICP score based on that data using Airtop.

- **Nodes Involved:**  
  - Extract Person Data  
  - ICP Scoring

- **Node Details:**

  - **Extract Person Data**  
    - *Type:* Execute Workflow  
    - *Role:* Invokes "AIRTOP — Extract LinkedIn Profile Information" sub-workflow to retrieve detailed LinkedIn data using the verified LinkedIn URL and Airtop_profile.  
    - *Input/Output:* Inputs LinkedIn_URL and Airtop_profile; outputs enriched person data to "Merge".  
    - *Potential Failures:* API errors, incomplete profile data, network issues.

  - **ICP Scoring**  
    - *Type:* Execute Workflow  
    - *Role:* Invokes "AIRTOP — Person ICP Scoring with Linkedin" sub-workflow to compute an Ideal Customer Profile score using the same LinkedIn URL and Airtop_profile.  
    - *Input/Output:* Inputs LinkedIn_URL and Airtop_profile; outputs scoring results to "Merge".  
    - *Potential Failures:* Scoring logic errors, API failures.

---

#### 2.5 Output Consolidation

- **Overview:**  
  Combines the enriched LinkedIn data and ICP score results into a single cohesive output object.

- **Nodes Involved:**  
  - Merge  
  - Sticky Note1 (Enrich Person Data)

- **Node Details:**

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Joins three input streams: enriched data from Extract Person Data, scoring data from ICP Scoring, and a fallback or initial source (likely the input or metadata).  
    - *Configuration:* Number of inputs set to 3; merges all incoming data.  
    - *Input/Output:* Inputs from Extract Person Data (index 0), Is valid Linkedin URL? (index 1), ICP Scoring (index 2); output is consolidated enriched profile and score.  
    - *Potential Failures:* Data shape mismatches, missing inputs if prior nodes fail.

  - **Sticky Note1**  
    - *Content:* "## Enrich Person Data\n** Find Linkedin Profile, extract all information and calculate ICP score"  
    - *Purpose:* Documentation clarifying the enrichment and scoring process.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                                         | Input Node(s)                  | Output Node(s)                              | Sticky Note                                                                                                    |
|-------------------------|-----------------------------|---------------------------------------------------------|-------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                | Receives user input via web form                         | -                             | Unify Parameters                            | Run this workflow using a form or from another workflow                                                        |
| When Executed by Another Workflow | Execute Workflow Trigger   | Receives inputs when called from another workflow        | -                             | Unify Parameters                            | Run this workflow using a form or from another workflow                                                        |
| Unify Parameters        | Set                         | Standardizes input parameter names                        | On form submission, When Executed by Another Workflow | Is corporate email?                        | Run this workflow using a form or from another workflow                                                        |
| Is corporate email?     | Filter                      | Validates corporate nature of email                       | Unify Parameters              | Find Person Linkedin URL                    |                                                                                                               |
| Find Person Linkedin URL | Execute Workflow            | Finds and verifies LinkedIn URL via Airtop               | Is corporate email?           | Is valid Linkedin URL?                      |                                                                                                               |
| Is valid Linkedin URL?  | Filter                      | Validates LinkedIn URL is non-empty and valid            | Find Person Linkedin URL      | Extract Person Data, ICP Scoring, Merge    |                                                                                                               |
| Extract Person Data     | Execute Workflow            | Extracts detailed LinkedIn profile data                   | Is valid Linkedin URL?        | Merge                                      | Find Linkedin Profile, extract all information and calculate ICP score                                         |
| ICP Scoring             | Execute Workflow            | Calculates Ideal Customer Profile score                   | Is valid Linkedin URL?        | Merge                                      | Find Linkedin Profile, extract all information and calculate ICP score                                         |
| Merge                   | Merge                       | Consolidates enriched data and ICP score                  | Extract Person Data, Is valid Linkedin URL?, ICP Scoring | -                                         | Find Linkedin Profile, extract all information and calculate ICP score                                         |
| Sticky Note             | Sticky Note                 | Documentation on input parameters                         | -                             | -                                           | Run this workflow using a form or from another workflow                                                        |
| Sticky Note1            | Sticky Note                 | Documentation on enrichment and scoring                   | -                             | -                                           | Find Linkedin Profile, extract all information and calculate ICP score                                         |
| Sticky Note7            | Sticky Note                 | Detailed README and usage notes for the entire workflow  | -                             | -                                           | README with use case, setup requirements, and next steps, including [Airtop API Key](https://portal.airtop.ai/api-keys) links |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Form Trigger** node named "On form submission".  
     - Configure form title: "Linkedin Profile Extractor".  
     - Add required fields: Person Name, Work Email, Airtop Profile (connected to Linkedin).  
     - Add form description with HTML content explaining the use and linking to https://portal.airtop.ai/browser-profiles.

   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
     - Define inputs: person_name, work_email, Airtop_profile.

2. **Create Parameter Unification Node:**

   - Add a **Set** node named "Unify Parameters".  
     - Add three string fields: Person_name, Work_email, Airtop_profile.  
     - Use expressions to populate each field from either form or workflow trigger input:  
       - Person_name: `{{$json["Person Name"] || $json.person_name}}`  
       - Work_email: `{{$json["Work Email"] || $json.work_email}}`  
       - Airtop_profile: `{{$json["Airtop Profile (connected to Linkedin)"] || $json.Airtop_profile}}`

3. **Create Corporate Email Validation Node:**

   - Add a **Filter** node named "Is corporate email?".  
     - Add multiple conditions combined with AND to exclude emails ending with or containing common free domains:  
       `@gmail.com`, `@qq.com`, `@hotmail.`, `@proton.me`, `@msn.`, `@yahoo.`, `@aol.`, `.edu`, `.net`.

4. **Create LinkedIn Profile Discovery and Validation:**

   - Add an **Execute Workflow** node named "Find Person Linkedin URL".  
     - Link this node to the sub-workflow "AIRTOP — LinkedIn Profile Discovery w Verification".  
     - Map inputs:  
       - Person_info: `={{ $json.Person_name }} - {{ $json.Work_email }}`  
       - Airtop_profile: `={{ $json.Airtop_profile }}`  

   - Add a **Filter** node named "Is valid Linkedin URL?".  
     - Conditions:  
       - `data.modelResponse` is not empty.  
       - `data.modelResponse` does not start with "NA".

5. **Add Data Enrichment and Scoring Nodes:**

   - Add an **Execute Workflow** node named "Extract Person Data".  
     - Link to sub-workflow "AIRTOP — Extract LinkedIn Profile Information".  
     - Map inputs:  
       - Linkedin_URL: `={{ $('Find Person Linkedin URL').item.json.data.modelResponse }}`  
       - Airtop_profile: `={{ $json.Airtop_profile }}`

   - Add an **Execute Workflow** node named "ICP Scoring".  
     - Link to sub-workflow "AIRTOP — Person ICP Scoring with Linkedin".  
     - Map inputs:  
       - Linkedin_URL: `={{ $('Find Person Linkedin URL').item.json.data.modelResponse }}`  
       - Airtop_profile: `={{ $json.Airtop_profile }}`

6. **Add Merge Node:**

   - Add a **Merge** node named "Merge".  
     - Set "Number of Inputs" to 3.  
     - Connect inputs from:  
       - Extract Person Data (input 0)  
       - Is valid Linkedin URL? (input 1)  
       - ICP Scoring (input 2)

7. **Connect Nodes:**

   - Connect "On form submission" and "When Executed by Another Workflow" both to "Unify Parameters".  
   - Connect "Unify Parameters" to "Is corporate email?".  
   - Connect "Is corporate email?" to "Find Person Linkedin URL".  
   - Connect "Find Person Linkedin URL" to "Is valid Linkedin URL?".  
   - Connect "Is valid Linkedin URL?" to "Extract Person Data", "ICP Scoring", and "Merge" as per inputs described.  
   - Connect "Extract Person Data" and "ICP Scoring" to "Merge".

8. **Add Sticky Notes for Documentation:**

   - Add sticky notes to document input parameters, enrichment process, and provide README information as per original.

9. **Configure Credentials:**

   - Ensure Airtop API credentials are set up and linked for all sub-workflows requiring them.  
   - Verify OAuth or API token setup for Airtop browser profiles.

10. **Sub-Workflows Setup:**

    - Ensure the following sub-workflows are imported or created:  
      - "AIRTOP — LinkedIn Profile Discovery w Verification"  
      - "AIRTOP — Extract LinkedIn Profile Information"  
      - "AIRTOP — Person ICP Scoring with Linkedin"  
    - Confirm each accepts and returns the parameters as mapped.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow uses Airtop profiles authenticated on LinkedIn to enrich contact data and calculate ICP scores. It requires an Airtop API key and profile configured with LinkedIn login.                                                                                                                                                                                                                                                                                                                                                                            | Airtop API Keys: https://portal.airtop.ai/api-keys                                                       |
| The input form includes a link to create or manage Airtop browser profiles: https://portal.airtop.ai/browser-profiles                                                                                                                                                                                                                                                                                                                                                                                                                                            | Airtop Profiles: https://portal.airtop.ai/browser-profiles                                               |
| The README sticky note includes detailed use case explanation, setup steps, and suggestions for next steps such as CRM integration and batch processing.                                                                                                                                                                                                                                                                                                                                                                                                          | Included in the Sticky Note7 node                                                                        |
| Filtering excludes common free email domains to focus on business emails for higher quality enrichment results. This may require updates to the filter list depending on specific use cases.                                                                                                                                                                                                                                                                                                                                                                        | Email Validation Logic                                                                                    |
| The workflow merges data from multiple sub-workflows, so any changes in those sub-workflows’ input/output schemas require corresponding updates here.                                                                                                                                                                                                                                                                                                                                                                                                          | Sub-workflow dependencies                                                                                 |

---

**Disclaimer:** The provided text is exclusively generated from an n8n automated workflow. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.