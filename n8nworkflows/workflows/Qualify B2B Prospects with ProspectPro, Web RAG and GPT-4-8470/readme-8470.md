Qualify B2B Prospects with ProspectPro, Web RAG and GPT-4

https://n8nworkflows.xyz/workflows/qualify-b2b-prospects-with-prospectpro--web-rag-and-gpt-4-8470


# Qualify B2B Prospects with ProspectPro, Web RAG and GPT-4

### 1. Workflow Overview

This workflow automates the qualification and segmentation of B2B prospects based on online content and AI analysis. It is designed primarily for companies using ProspectPro and Bedrijfsdata.nl services to enrich prospect data with live web and search content, analyze it with GPT-4 via LangChain integration, and update prospect records accordingly.

The workflow logically divides into these functional blocks:

- **1.1 Input Reception & Initial Checks:** Receive a prospect ID trigger, retrieve prospect details, and verify if the prospect has been processed before to avoid duplicate processing.
- **1.2 Web and Search Content Retrieval:** Using the prospect’s domain, collect relevant website and search engine content through RAG (Retrieval-Augmented Generation) APIs.
- **1.3 Content Processing & Qualification with AI:** Preprocess retrieved content and feed it into an LLM chain (GPT-4) to classify the company and determine qualification tags.
- **1.4 Prospect Update & Tagging:** Based on AI output and business logic, tag, label, and update the prospect record in ProspectPro.
- **1.5 Workflow Outcome & Error Handling:** Define output for parent workflows and handle cases requiring manual qualification or error logging.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initial Checks

- **Overview:** This block triggers the workflow with a prospect ID, fetches full prospect details, and checks if the prospect was already qualified or manually tagged, thus preventing redundant processing.

- **Nodes Involved:**  
  - ProspectPro Trigger Example  
  - When Executed by Another Workflow  
  - Get Prospect  
  - Not Processed Before?  
  - Continue?

- **Node Details:**

  - **ProspectPro Trigger Example**  
    - Type: ProspectPro trigger node  
    - Configuration: Polls every minute for new prospects with label -1 and excludes those already tagged "AutoQualified" or "ManualQualification".  
    - Inputs: None (start node)  
    - Outputs: Prospect ID to "When Executed by Another Workflow"  
    - Edge cases: May trigger multiple times if tags not updated; relies on correct tag management.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows external workflows to trigger this workflow by passing a prospect ID.  
    - Inputs: Prospect ID in workflow inputs  
    - Outputs: Prospect ID to "Get Prospect"

  - **Get Prospect**  
    - Type: ProspectPro API node  
    - Role: Retrieves full prospect data by ID from ProspectPro.  
    - Configuration: Uses ProspectPro API credentials; fetches by dynamic ID from trigger node.  
    - Outputs: Prospect JSON to "Not Processed Before?"  
    - Edge cases: API failures, missing prospect ID, or incomplete data.

  - **Not Processed Before?**  
    - Type: If node  
    - Role: Checks if prospect tags include "AutoQualified" or "ManualQualification".  
    - Condition: Returns true if these tags are absent (not processed); false otherwise.  
    - Outputs:  
      - True: proceeds to "Continue?"  
      - False: ends with returning {result: false} to stop processing.  
    - Edge cases: Tag format inconsistencies, empty or missing tags.

  - **Continue?**  
    - Type: If node  
    - Role: Checks if the prospect has a valid domain to proceed with web content retrieval.  
    - Condition: Domain field is non-empty.  
    - Outputs:  
      - True: goes to "Get RAG domain"  
      - False: routes to manual qualification tagging.  
    - Edge cases: Missing or invalid domain may cause fallback to manual processing.

---

#### 2.2 Web and Search Content Retrieval

- **Overview:** This block gathers live website content and search engine snippets relevant to the prospect’s domain using Bedrijfsdata.nl RAG APIs.

- **Nodes Involved:**  
  - Get RAG domain  
  - Get RAG search  
  - Process live web content  
  - Content available to proceed?

- **Node Details:**

  - **Get RAG domain**  
    - Type: Bedrijfsdata RAG API node  
    - Role: Retrieves domain-specific web content and metadata.  
    - Configuration: Uses the prospect domain dynamically; requires Bedrijfsdata API credentials.  
    - Output: JSON containing website texts and company info.  
    - Edge cases: API errors, rate limits, domain not found.

  - **Get RAG search**  
    - Type: Bedrijfsdata RAG API node  
    - Role: Retrieves search engine snippets related to the prospect.  
    - Configuration: Uses a query combining company name and domain; limits to 10 results.  
    - Edge cases: No results, API errors.

  - **Process live web content**  
    - Type: Code node  
    - Role: Extracts and structures relevant data from RAG domain and search results into fields like keywords, descriptions, about, and snippets.  
    - Key expressions: Filters and maps search results, merges data into a structured JSON for AI input.  
    - Edge cases: Missing data in either RAG source results in null fields, handled gracefully.

  - **Content available to proceed?**  
    - Type: If node  
    - Role: Checks if either website content or search content is available post-processing.  
    - Outputs:  
      - True: continue to AI qualification  
      - False: fallback to manual qualification tagging.

---

#### 2.3 Content Processing & Qualification with AI

- **Overview:** This block utilizes an LLM chain (GPT-4) to analyze structured web content and classify the company according to business-relevant criteria.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Valid results?

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain node  
    - Role: Prepares and sends a prompt to the OpenAI GPT-4 model, providing structured company data and web content.  
    - Configuration: Uses Dutch language instructions, specifies JSON output schema, combines multiple data fields into a prompt.  
    - Input: JSON from "Process live web content"  
    - Output: Raw AI response to "Structured Output Parser"  
    - Edge cases: Token limits, incomplete data, LLM errors.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI GPT-4 node  
    - Role: Executes the GPT-4 model with a 4000 token limit.  
    - Configuration: Model version "gpt-4.1-mini" selected, OpenAI API credentials required.  
    - Input: Prompt from "Basic LLM Chain"  
    - Output: AI response to "Basic LLM Chain"  
    - Edge cases: API rate limits, timeout, authentication failure.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser node  
    - Role: Parses AI response strictly according to a JSON schema defining classification booleans (is_b2b, is_marketing, is_sales, is_crm_expert, is_saas).  
    - Configuration: Manual JSON schema enforcing required boolean outputs, disallowing additional properties.  
    - Output: Parsed structured data to "Basic LLM Chain" for validation.  
    - Edge cases: Parsing failure if AI output is malformed, schema mismatches.

  - **Valid results?**  
    - Type: If node  
    - Role: Checks if the parsed AI output is valid and contains the expected keys, especially "is_b2b".  
    - Outputs:  
      - True: proceed to qualification and tagging  
      - False: fallback to manual qualification tagging  
    - Notes: Although redundant due to parser errors, it allows customized validation logic.

---

#### 2.4 Prospect Update & Tagging

- **Overview:** This block applies business rules to classify the prospect based on AI output and existing prospect data, then updates tags and labels on the ProspectPro platform.

- **Nodes Involved:**  
  - Qualify & Tag Prospect  
  - Update Prospect in ProspectPro  
  - Set Tag: ManualQualification  
  - Update prospect

- **Node Details:**

  - **Qualify & Tag Prospect**  
    - Type: Code node  
    - Role: Implements custom logic to assign labels and tags based on AI classification and competitor apps present in the prospect data.  
    - Logic highlights:  
      - Label 1 for B2B companies fitting ICP (Ideal Customer Profile)  
      - Tags "Potential API", "Potential ProspectPro", or "Potential Dataset" based on SaaS, CRM expertise, or competitor presence  
      - Always appends "AutoQualified" tag to mark processing.  
    - Outputs: Updated prospect object with modified tags and label.  
    - Edge cases: Empty apps or tags arrays handled; logic intended for customization.

  - **Update Prospect in ProspectPro**  
    - Type: ProspectPro API node  
    - Role: Sends patch requests to update tags and label fields in ProspectPro for the given prospect ID.  
    - Uses credentials for ProspectPro API.  
    - Outputs: Updated prospect JSON to "return { result: prospect.label === 1 };"  
    - Edge cases: API errors, partial updates, network issues.

  - **Set Tag: ManualQualification**  
    - Type: Code node  
    - Role: Adds the tag "ManualQualification" to prospects for which automated qualification is not possible or fails, marking them for manual review.  
    - Output: Prospect data with updated tags.  
    - Edge cases: Ensures tags array exists; no label changes here.

  - **Update prospect**  
    - Type: ProspectPro API node  
    - Role: Updates prospect tags in ProspectPro for manual qualification cases or error handling.  
    - Edge cases: API errors, consistency in tag updates.

---

#### 2.5 Workflow Outcome & Error Handling

- **Overview:** This block defines the workflow's final output for parent workflows and handles error logging or fallback results.

- **Nodes Involved:**  
  - return { result: prospect.label === 1 };  
  - return { result: true };  
  - return { result: false };

- **Node Details:**

  - **return { result: prospect.label === 1 };**  
    - Type: Code node  
    - Role: Outputs a boolean indicating if the prospect was qualified (label 1).  
    - Used by parent workflows to decide next steps.

  - **return { result: true };**  
    - Type: Code node  
    - Role: Returns true unconditionally, used when qualification is positive or for test flow.  

  - **return { result: false };**  
    - Type: Code node  
    - Role: Returns false unconditionally, used when disqualification or manual qualification is required.

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                         | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                   |
|-----------------------------|----------------------------------|---------------------------------------|--------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| ProspectPro Trigger Example  | ProspectPro Trigger              | Workflow trigger for new prospects    | None                           | When Executed by Another Workflow | See Sticky Note6 for workflow description and usage                                         |
| When Executed by Another Workflow | Execute Workflow Trigger      | External workflow trigger by ID       | ProspectPro Trigger Example     | Get Prospect                    |                                                                                              |
| Get Prospect                | ProspectPro API                  | Fetch full prospect data by ID        | When Executed by Another Workflow | Not Processed Before?          | See Sticky Note9 for start logic and domain importance                                      |
| Not Processed Before?       | If                              | Check if prospect was already processed | Get Prospect                   | Continue? / return { result: false } |                                                                                              |
| Continue?                  | If                              | Check if domain is available           | Not Processed Before?           | Get RAG domain / Set Tag: ManualQualification |                                                                                              |
| Get RAG domain             | Bedrijfsdata RAG API             | Retrieve domain-specific web content  | Continue?                      | Get RAG search                  | See Sticky Note10 for web and search content retrieval                                       |
| Get RAG search             | Bedrijfsdata RAG API             | Retrieve search engine snippets        | Get RAG domain                  | Process live web content        |                                                                                              |
| Process live web content    | Code                            | Extract and structure web/search data | Get RAG search                 | Content available to proceed?   |                                                                                              |
| Content available to proceed? | If                            | Check if web or search content exists | Process live web content        | Basic LLM Chain / Set Tag: ManualQualification |                                                                                              |
| Basic LLM Chain            | LangChain LLM Chain              | Prepare prompt and send to GPT-4      | Content available to proceed?   | Valid results?                  | See Sticky Note8 for LLM usage and prompt instructions                                      |
| OpenAI Chat Model          | LangChain OpenAI GPT-4           | Execute GPT-4 model                    | Basic LLM Chain                | Basic LLM Chain                 |                                                                                              |
| Structured Output Parser    | LangChain Output Parser          | Parse AI response to defined JSON schema | Basic LLM Chain               | Basic LLM Chain                 |                                                                                              |
| Valid results?             | If                              | Validate AI output schema correctness | Basic LLM Chain                | Qualify & Tag Prospect / Set Tag: ManualQualification | See Sticky Note (validation note)                                                          |
| Qualify & Tag Prospect      | Code                            | Apply business logic for labels/tags  | Valid results?                 | Update Prospect in ProspectPro / Set Tag: ManualQualification | See Sticky Note7 for qualification logic guidance                                         |
| Update Prospect in ProspectPro | ProspectPro API               | Update prospect tags and label        | Qualify & Tag Prospect          | return { result: prospect.label === 1 } / Set Tag: ManualQualification |                                                                                              |
| Set Tag: ManualQualification | Code                           | Add manual qualification tag           | Multiple fallback points       | Update prospect                | See Sticky Note5 for error logging advice                                                  |
| Update prospect             | ProspectPro API                  | Update prospect tags for manual cases | Set Tag: ManualQualification   | return { result: false }         |                                                                                              |
| return { result: prospect.label === 1 } | Code                  | Output final qualification result      | Update Prospect in ProspectPro | None                          | See Sticky Note4 for workflow output description                                           |
| return { result: true }     | Code                            | Output positive result                  | Is Qualified?                  | None                          |                                                                                              |
| return { result: false }    | Code                            | Output negative result                  | Multiple nodes                | None                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **ProspectPro Trigger** node:
     - Configure to poll every minute for prospects with label -1.
     - Exclude prospects tagged "AutoQualified" or "ManualQualification".
   - Add an **Execute Workflow Trigger** named "When Executed by Another Workflow":
     - Accept input parameter "id" (prospect ID).

2. **Get Prospect Data:**
   - Add a **ProspectPro API** node "Get Prospect":
     - Operation: Get prospect by ID.
     - Use the prospect ID from trigger nodes.
     - Configure credentials for ProspectPro API.

3. **Check Processing Status:**
   - Add an **If** node "Not Processed Before?":
     - Condition: Check if tags array does NOT include "AutoQualified" or "ManualQualification".
     - True branch continues; False branch returns `{ result: false }` (end workflow).

4. **Domain Check:**
   - Add an **If** node "Continue?":
     - Condition: Check if the `domain` field is non-empty.
     - True branch proceeds; False branch adds manual qualification tag.

5. **Retrieve Web Content via RAG:**
   - Add a **Bedrijfsdata RAG API** node "Get RAG domain":
     - Operation: get_domain.
     - Input: prospect domain.
     - Use Bedrijfsdata API credentials.
   - Add another **Bedrijfsdata RAG API** node "Get RAG search":
     - Operation: get_search.
     - Query: combine prospect name and domain.
     - Limit to 10 results.

6. **Process Web and Search Content:**
   - Add a **Code** node "Process live web content":
     - Extract and structure website keywords, descriptions, about info, and search snippets.
     - Handle missing data gracefully.
   - Add an **If** node "Content available to proceed?":
     - Condition: Check if website_content or search_content exists.
     - True proceeds to AI; False adds manual qualification tag.

7. **AI Qualification:**
   - Add a **LangChain LLM Chain** node "Basic LLM Chain":
     - Prepare prompt with structured company info and web content.
     - Include instructions for Dutch output and JSON structure.
     - Connect to OpenAI GPT-4 model node.
   - Add a **LangChain OpenAI** node "OpenAI Chat Model":
     - Model: gpt-4.1-mini.
     - Token limit: 4000.
     - Use OpenAI API credentials.
   - Add a **LangChain Output Parser** node:
     - Define JSON schema for company classification booleans.
   - Add an **If** node "Valid results?":
     - Condition: Check parsed output has keys and especially "is_b2b".
     - True continues; False tags for manual qualification.

8. **Prospect Qualification and Tagging:**
   - Add a **Code** node "Qualify & Tag Prospect":
     - Implement logic:
       - Label 1 if is_b2b true.
       - Add tags based on is_saas, is_crm_expert, competitor apps.
       - Always add "AutoQualified" tag.
   - Add a **ProspectPro API** node "Update Prospect in ProspectPro":
     - Operation: patch.
     - Update tags and label.
   - Add a **Code** node to output `{ result: prospect.label === 1 }`.

9. **Manual Qualification Fallback:**
   - Add a **Code** node "Set Tag: ManualQualification":
     - Append "ManualQualification" tag.
   - Add a **ProspectPro API** node "Update prospect":
     - Patch tags only.
   - Add a **Code** node to output `{ result: false }`.

10. **Final Outcome Handling:**
    - Add code nodes returning `{ result: true }` or `{ result: false }` as appropriate after qualification checks.

11. **Connect all nodes according to described flow and conditions.**

12. **Credential Setup:**
    - ProspectPro API credentials with read/write permissions.
    - Bedrijfsdata API credentials for RAG calls.
    - OpenAI API credentials for GPT-4 access.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow outputs `{ result: Boolean }` to allow parent workflows to decide next steps based on qualification.    | Sticky Note4                                                   |
| Tags "AutoQualified" and "ManualQualification" prevent duplicate processing and mark workflow status.           | Sticky Note6, Sticky Note9                                     |
| Use LangChain’s Structured Output Parser with strict JSON schema to enforce AI response format and quality.     | Sticky Note8                                                  |
| Customize the qualification business logic in the "Qualify & Tag Prospect" node to fit your ICP and use case.   | Sticky Note7                                                  |
| Log errors by tagging prospects accordingly and monitor for manual intervention needs.                           | Sticky Note5                                                  |
| For support and best practices, contact https://www.prospectpro.nl/klantenservice/                               | Sticky Note6                                                  |
| The workflow can be triggered directly by ProspectPro events or externally by other workflows passing prospect IDs. | Workflow Overview, Sticky Note9                               |

---

This reference document fully describes the structure, logic, configuration, and integration points of the "Qualify B2B Prospects with ProspectPro, Web RAG and GPT-4" workflow, allowing full understanding, reproduction, and extension.