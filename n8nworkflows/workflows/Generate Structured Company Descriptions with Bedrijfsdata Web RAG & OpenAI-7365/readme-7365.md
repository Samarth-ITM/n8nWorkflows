Generate Structured Company Descriptions with Bedrijfsdata Web RAG & OpenAI

https://n8nworkflows.xyz/workflows/generate-structured-company-descriptions-with-bedrijfsdata-web-rag---openai-7365


# Generate Structured Company Descriptions with Bedrijfsdata Web RAG & OpenAI

### 1. Workflow Overview

This workflow, titled **"Generate Structured Company Descriptions with Bedrijfsdata Web RAG & OpenAI"**, is designed to create synthetic, structured company descriptions by combining live web content retrieval (via RAG from Bedrijfsdata.nl services) with advanced language model processing (OpenAI GPT models). It targets use cases where enhanced, human-readable company profiles are needed for applications such as CRM enrichment, sales prospect qualification, or marketing content generation.

The workflow is logically divided into four major blocks:

- **1.1 Input Reception & Validation**: Receives a company identifier or domain and verifies that a valid domain is present.
- **1.2 Data Retrieval (Web RAG)**: Uses the Bedrijfsdata.nl API to retrieve live domain-based content and search engine snippets related to the company.
- **1.3 Content Preparation & AI Processing**: Prepares collected data into structured JSON and feeds it into a language model chain with a defined prompt and structured output parser to generate synthetic company descriptions.
- **1.4 Output Handling & Error Management**: Updates the company's description in HubSpot CRM and includes nodes for error handling at each critical step.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block initiates the workflow by receiving an input (typically a company ID), fetching company data, and validating that a domain name is available before proceeding.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Get prospect  
  - Company domain is required (IF)  
  - Error type 1: invalid input (NoOp)  
  - Sticky Note (step 1)

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Trigger node to start workflow via external execution.  
    - Config: Accepts input parameter "id" (company ID).  
    - Inputs/Outputs: Output passes company ID to "Get prospect".  
    - Edge Cases: Missing or malformed "id" input could halt flow.

  - **Get prospect**  
    - Type: Bedrijfsdata.nl ProspectPro API node.  
    - Config: Retrieves company data by provided ID.  
    - Credentials: ProspectPro API with Bedrijfsdata.nl account.  
    - Inputs: Receives "id" from trigger.  
    - Outputs: Company JSON including domain, city, country_code, SBI code, and optionally Google industry.  
    - Edge Cases: API failure or missing company data.

  - **Company domain is required** (IF node)  
    - Type: Conditional check.  
    - Config: Verifies that the "domain" field in the JSON is truthy (exists and non-empty).  
    - Inputs: From "Get prospect".  
    - Outputs:  
      - True branch leads to data retrieval nodes.  
      - False branch leads to "Error type 1: invalid input".  
    - Edge Cases: Domain missing or empty string blocks further processing.

  - **Error type 1: invalid input**  
    - Type: No operation node to mark input validation failure.  
    - Role: Placeholder to handle or log invalid input scenarios.

  - **Sticky Note**  
    - Content: Explains the requirement for a domain name as starting input and warns about node references when modifying.

---

#### 2.2 Data Retrieval (Web RAG)

- **Overview:**  
  This block collects live web content and search snippets related to the company domain using Bedrijfsdata.nl's RAG APIs. It includes two sources: domain content and search engine snippets.

- **Nodes Involved:**  
  - Get RAG domain  
  - Get rag search  
  - Prepare content to feed to LLM (Code)  
  - Content available to proceed? (IF)  
  - Error type 2: RAG error (NoOp)  
  - Sticky Note1 (step 2)

- **Node Details:**

  - **Get RAG domain**  
    - Type: Bedrijfsdata.nl RAG domain API node.  
    - Config: Queries live web content for the company's domain.  
    - Inputs: Receives domain from "Company domain is required".  
    - Credentials: Bedrijfsdata.nl API key (ProspectPro account).  
    - Outputs: JSON including website text snippets, meta articles, keywords, descriptions.  
    - Edge Cases: API failure, SEO-unfriendly websites causing empty or incomplete content.

  - **Get rag search**  
    - Type: Bedrijfsdata.nl RAG search API node.  
    - Config: Queries search engine results related to company name and domain.  
    - Inputs: From "Get RAG domain" (sequential).  
    - Credentials: Same Bedrijfsdata.nl API key.  
    - Outputs: Search results with snippets and descriptions.  
    - Edge Cases: Search API failure or no relevant results.

  - **Prepare content to feed to LLM** (Code node)  
    - Type: JavaScript code execution.  
    - Role: Merges and filters data from prospect, RAG domain, and RAG search into structured JSON for LLM input.  
    - Key Logic:  
      - Extracts basic company info (name, city, country_code, domain, sbi, google_industry).  
      - Extracts website content fields (keywords, descriptions, about, website snippets) if available.  
      - Extracts filtered search results with URLs, descriptions, snippets.  
    - Inputs: JSON from "Get prospect", "Get RAG domain", and "Get rag search".  
    - Outputs: Combined structured JSON.  
    - Edge Cases: Missing or error fields in RAG data handled by null assignment.

  - **Content available to proceed?** (IF node)  
    - Type: Conditional check.  
    - Config: Proceeds only if either website_content or search_content is non-empty/truthy.  
    - Inputs: From "Prepare content to feed to LLM".  
    - Outputs:  
      - True branch proceeds to LLM processing.  
      - False branch leads to "Error type 2: RAG error".  
    - Edge Cases: No content found, blocks further LLM processing.

  - **Error type 2: RAG error**  
    - Type: No operation node.  
    - Role: Placeholder for handling errors related to content retrieval.

  - **Sticky Note1**  
    - Content: Describes the use of two RAG services, filtering content, token usage considerations, and SEO-friendliness warnings. Includes links to Bedrijfsdata.nl developer platform.

---

#### 2.3 Content Preparation & AI Processing

- **Overview:**  
  This block uses the prepared content to prompt a language model (OpenAI GPT-4.1-mini) with a tailored prompt and a structured output parser to generate a detailed, structured synthetic company description.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Error type 3: LLM error (NoOp)  
  - Sticky Note2 (step 3)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain node in n8n.  
    - Role: Defines the prompt template and manages the LLM interaction with output parsing.  
    - Config:  
      - Prompt includes placeholders for company info, website descriptions, about, search snippets.  
      - Instructions emphasize: Dutch output, JSON structure adherence, combining data engineer input with general knowledge.  
      - Has output parser enabled to enforce JSON structured output.  
    - Inputs: From "Content available to proceed?" (truthy branch).  
    - Outputs: Parsed JSON with keys: "company_description", "products_and_services", "target_audience".  
    - Edge Cases: Prompt or parsing errors, token limit issues.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat model node.  
    - Config: Uses model "gpt-4.1-mini" with max tokens 4000.  
    - Credentials: OpenAI API key.  
    - Inputs: Receives messages from "Basic LLM Chain".  
    - Outputs: Raw LLM response to be parsed.  
    - Edge Cases: API failures, rate limits, timeouts.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser node.  
    - Config: Specifies expected JSON schema example for company_description, products_and_services, and target_audience.  
    - Inputs: Raw LLM output from "OpenAI Chat Model".  
    - Outputs: Parsed JSON enforcing the schema.  
    - Edge Cases: Parsing failures if output deviates from expected schema.

  - **Error type 3: LLM error**  
    - Type: No operation node.  
    - Role: Placeholder for handling errors in LLM processing.

  - **Sticky Note2**  
    - Content: Explains prompt setup, token limits, RAG approach benefits, and recommendations for complex use cases.

---

#### 2.4 Output Handling & Error Management

- **Overview:**  
  This block updates the HubSpot CRM company record with the generated synthetic description and contains nodes to handle potential errors at various stages.

- **Nodes Involved:**  
  - Update company description (HubSpot node)  
  - Error type 4: Hubspot error (NoOp)  
  - Sticky Note3 (step 4)  
  - Sticky Note7 (error handling guidelines)

- **Node Details:**

  - **Update company description**  
    - Type: HubSpot node to update company resource.  
    - Config:  
      - Updates "description" field with the generated "company_description" JSON output.  
      - Uses OAuth2 authentication with HubSpot.  
      - Company ID taken from initial workflow trigger input.  
    - Inputs: Output from "Basic LLM Chain".  
    - Outputs: None on success; error branch to "Error type 4: Hubspot error".  
    - Edge Cases: OAuth authentication failures, API errors, invalid company ID.

  - **Error type 4: Hubspot error**  
    - Type: No operation node.  
    - Role: Placeholder for handling HubSpot update errors.

  - **Sticky Note3**  
    - Content: Suggestions on how to use the synthetic content (CRM enrichment, manual evaluation, automation, etc.).

  - **Sticky Note7**  
    - Content: General advice on implementing error handling such as logging errors to Google Sheets or sending notifications via Slack, Telegram, or email.

---

### 3. Summary Table

| Node Name                     | Node Type                                      | Functional Role                          | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                                      |
|-------------------------------|------------------------------------------------|----------------------------------------|-----------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger                        | Entry point, receives company ID       |                             | Get prospect                      |                                                                                                                  |
| Get prospect                   | Bedrijfsdata.nl ProspectPro API                 | Fetch company data by ID                | When Executed by Another Workflow | Company domain is required        |                                                                                                                  |
| Company domain is required     | IF Condition                                    | Validates presence of domain            | Get prospect                 | Get RAG domain, Error type 1      | Step 1: requires a domain name; adjust with care due to node references                                         |
| Error type 1: invalid input    | No Operation                                    | Handles invalid/missing domain input    | Company domain is required (false) |                               |                                                                                                                  |
| Get RAG domain                | Bedrijfsdata.nl RAG Domain API                   | Retrieves live domain content           | Company domain is required (true) | Get rag search                    | Step 2: retrieves live web content using Bedrijfsdata.nl RAG; SEO-friendly sites required; see developer links    |
| Get rag search                | Bedrijfsdata.nl RAG Search API                   | Retrieves search engine snippets        | Get RAG domain               | Prepare content to feed to LLM    |                                                                                                                  |
| Prepare content to feed to LLM | Code                                            | Prepares and merges content for LLM    | Get rag search, Get prospect, Get RAG domain | Content available to proceed?     |                                                                                                                  |
| Content available to proceed?  | IF Condition                                    | Checks if sufficient content is present | Prepare content to feed to LLM | Basic LLM Chain, Error type 2     |                                                                                                                  |
| Error type 2: RAG error        | No Operation                                    | Handles errors in content retrieval     | Content available to proceed? (false) |                               |                                                                                                                  |
| Basic LLM Chain               | LangChain LLM Chain                              | Defines prompt, manages LLM input/output | Content available to proceed? (true) | Update company description, Error type 3 | Step 3: create synthetic content; prompt configured for Dutch, JSON output; watch token limits                   |
| OpenAI Chat Model             | LangChain OpenAI Chat Model                      | Executes OpenAI GPT call                 | Basic LLM Chain (lmChatOpenAi input) | Basic LLM Chain (ai_languageModel output) |                                                                                                                  |
| Structured Output Parser       | LangChain Output Parser                          | Parses LLM output to JSON                | OpenAI Chat Model            | Basic LLM Chain (ai_outputParser output) |                                                                                                                  |
| Error type 3: LLM error       | No Operation                                    | Handles LLM processing errors            | Basic LLM Chain (error branch) |                               |                                                                                                                  |
| Update company description    | HubSpot Company Update Node                      | Updates HubSpot company description     | Basic LLM Chain              | Error type 4                    | Step 4: use synthetic content in CRM or other systems                                                          |
| Error type 4: Hubspot error    | No Operation                                    | Handles HubSpot API update failures     | Update company description (error) |                               |                                                                                                                  |
| Sticky Note                   | Sticky Note                                     | Instructional note                      |                             |                                   | Step 1: domain input explanation                                                                                 |
| Sticky Note1                  | Sticky Note                                     | Instructional note                      |                             |                                   | Step 2: RAG web content retrieval info, links to Bedrijfsdata.nl                                                |
| Sticky Note2                  | Sticky Note                                     | Instructional note                      |                             |                                   | Step 3: LLM prompt and token usage advice                                                                        |
| Sticky Note3                  | Sticky Note                                     | Instructional note                      |                             |                                   | Step 4: suggestions for using synthetic content                                                                 |
| Sticky Note4                  | Sticky Note                                     | Project overview and usage notes       |                             |                                   | Full workflow explanation and use cases, links to developer platform and LinkedIn                               |
| Sticky Note7                  | Sticky Note                                     | Error handling best practices          |                             |                                   | Advises logging and alerting strategies for error management                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure it to accept input parameter `id` (company ID).

2. **Fetch Company Data**  
   - Add **Bedrijfsdata.nl ProspectPro** node named `Get prospect`.  
   - Set operation to `get` company by `id` from previous node (`={{ $json.id }}`).  
   - Configure credentials with your Bedrijfsdata.nl ProspectPro API key.

3. **Validate Domain Presence**  
   - Add **IF** node named `Company domain is required`.  
   - Condition: Check if `{{ !!$json.domain }}` is true (non-empty domain).  
   - Connect `Get prospect` output to this node.

4. **Handle Missing Domain**  
   - Add **No Operation** node named `Error type 1: invalid input` connected to false branch of the IF.

5. **Retrieve Website Content via RAG Domain API**  
   - Add **Bedrijfsdata.nl Bedrijfsdata API** node named `Get RAG domain`.  
   - Set resource: `rag`, operation: `get_domain`.  
   - Parameter: domain from JSON `{{ $json.domain }}`.  
   - Use Bedrijfsdata.nl API credentials.  
   - Connect true branch of domain IF node to this node.

6. **Retrieve Search Snippets via RAG Search API**  
   - Add **Bedrijfsdata.nl Bedrijfsdata API** node named `Get rag search`.  
   - Set resource: `rag`, operation: `get_search`.  
   - Query: Combine company name and domain, e.g., `{{ $json.name }} {{ $json.domain }}`.  
   - Limit results to 10.  
   - Use same API credentials.  
   - Connect `Get RAG domain` output to this node.

7. **Prepare Content for LLM**  
   - Add **Code** node named `Prepare content to feed to LLM`.  
   - Mode: run once for each item.  
   - Paste the JavaScript code that merges prospect data with RAG domain and search results, extracting key fields (`basic_information`, `website_content`, `search_content`).  
   - Connect `Get rag search` output to this node.

8. **Check Content Availability**  
   - Add **IF** node named `Content available to proceed?`.  
   - Condition: Check if either `website_content` or `search_content` is truthy: `{{ !!$json.website_content || !!$json.search_content }}`.  
   - Connect `Prepare content to feed to LLM` output to this node.

9. **Handle Missing Content**  
   - Add **No Operation** node named `Error type 2: RAG error` on the false branch of the IF.

10. **Configure LLM Chain for Synthetic Content**  
    - Add **LangChain Chain LLM** node named `Basic LLM Chain`.  
    - Set prompt with placeholders for company info and contents as per the example, instructing the model to output Dutch textual values adhering to a JSON structure.  
    - Enable structured output parser.  
    - Connect true branch of `Content available to proceed?` node to this node.

11. **Add OpenAI Chat Model Node**  
    - Add **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.  
    - Choose model `gpt-4.1-mini`.  
    - Set max tokens to 4000.  
    - Configure OpenAI API credentials.  
    - Connect `Basic LLM Chain` node to this node's `ai_languageModel` input.

12. **Add Structured Output Parser Node**  
    - Add **LangChain Structured Output Parser** node named `Structured Output Parser`.  
    - Define the JSON schema example with keys: `company_description`, `products_and_services`, `target_audience`.  
    - Connect from `OpenAI Chat Model` node to this node's `ai_outputParser` input.  
    - Connect output of this parser back into `Basic LLM Chain` node.

13. **Handle LLM Errors**  
    - Add **No Operation** node named `Error type 3: LLM error` connected to the error output of `Basic LLM Chain`.

14. **Update HubSpot Company Description**  
    - Add **HubSpot** node named `Update company description`.  
    - Operation: `update` company.  
    - Company ID: `={{ $json.id }}` from trigger node.  
    - Update field: set `description` to `={{ $json.output.company_description }}` from LLM output.  
    - Use OAuth2 credentials for HubSpot.  
    - Connect success output of `Basic LLM Chain` to this node.

15. **Handle HubSpot Errors**  
    - Add **No Operation** node named `Error type 4: Hubspot error` connected to the error output of HubSpot node.

16. **Add Sticky Notes**  
    - Create sticky notes explaining each major step (input requirement, data retrieval, LLM prompt, usage, error handling) and place them appropriately on the canvas.

17. **Set Execution Order**  
    - Make sure connections follow the logical flows described.  
    - Enable error branches for monitoring and future handling.

18. **Test Workflow**  
    - Trigger with valid company ID and ensure domain is present.  
    - Verify data retrieval, LLM processing, and HubSpot update steps complete successfully.  
    - Monitor error nodes for graceful handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                          | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow only requires a domain name as input but starts from a company ID for richer data. You can alter the trigger to accept just domain names or other identifiers.                         | Workflow design note                                                  |
| Bedrijfsdata.nl Developer Platform provides APIs for live web data extraction (RAG) and prospect data. API docs: https://developers.bedrijfsdata.nl and https://www.bedrijfsdata.nl              | API documentation                                                    |
| The prompt and output parser are set to produce Dutch textual content adhering strictly to JSON output schemas to ensure downstream usability and parsing reliability.                               | Prompt design guideline                                              |
| The RAG approach allows use of less expensive LLM models (e.g., gpt-3.5-turbo or gpt-4.1-mini) while producing rich content by feeding them curated web data.                                      | Performance & cost optimization                                      |
| Always implement error handling, such as logging to Google Sheets or messaging via Slack/Telegram/email, for production robustness.                                                                  | Recommended best practice for error management                        |
| Share your results or customizations on LinkedIn with Bedrijfsdata.nl community: https://www.linkedin.com/company/bedrijfsdata-nl/                                                                   | Community engagement                                                 |
| Images illustrating example outputs are available on Bedrijfsdata.nl pages linked in the workflow sticky notes.                                                                                      | Visual examples                                                      |

---

This documentation provides a detailed and actionable reference to understand, reproduce, and extend the workflow for generating structured company descriptions using web RAG data and OpenAI LLMs.