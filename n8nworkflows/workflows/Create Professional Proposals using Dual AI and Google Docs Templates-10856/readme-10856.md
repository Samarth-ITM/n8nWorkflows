Create Professional Proposals using Dual AI and Google Docs Templates

https://n8nworkflows.xyz/workflows/create-professional-proposals-using-dual-ai-and-google-docs-templates-10856


# Create Professional Proposals using Dual AI and Google Docs Templates

### 1. Workflow Overview

This workflow automates the generation of professional business proposals by leveraging dual AI models and Google Docs templates. It transforms basic user inputs into a polished, readable proposal document, ready for immediate use or sharing.

The workflow is structured into the following logical blocks:

- **1.1 Manual Input Reception:** Collects core proposal parameters manually triggered by the user.
- **1.2 Initial Proposal Generation:** Uses OpenAI’s GPT-4o-mini model to expand and structure the input into a detailed proposal draft in JSON format.
- **1.3 Proposal Rewriting and Refinement:** Utilizes OpenRouter’s Claude 3.5 model to rewrite and polish the draft for clarity and professional tone.
- **1.4 Google Docs Template Copying:** Copies a predefined Google Docs template to create a new editable proposal document.
- **1.5 Variable Replacement in Google Docs:** Replaces template placeholders in the copied document with the refined proposal content.
- **1.6 Output Delivery:** Provides the final, updated Google Docs proposal document for user access.

This modular design ensures clear separation of concerns, facilitates troubleshooting, and enhances maintainability.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Input Reception

- **Overview:**  
  This block gathers the initial proposal parameters via a manual trigger node and sets predefined values representing client and project details.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set proposal values (Set Node)  

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - *Type & Role:* Manual trigger to start the workflow on demand.  
    - *Configuration:* No parameters; simply triggers the flow.  
    - *Connections:* Output → Set proposal values.  
    - *Failure Modes:* None specific; user must trigger manually.

  - **Set proposal values**  
    - *Type & Role:* Assigns static values simulating user input for proposal details.  
    - *Configuration:* Sets strings such as company name ("Google"), client name ("Sundar Pichai"), problem, solution, timeline, scope, and price.  
    - *Variables:* Uses fixed string assignments; no expressions.  
    - *Connections:* Input ← Manual trigger; Output → Create proposal node.  
    - *Edge Cases:* Manual editing needed for customization; no validation on input formats.

---

#### 2.2 Initial Proposal Generation

- **Overview:**  
  This block sends the collected input data to OpenAI’s GPT-4o-mini model to generate a structured, detailed proposal draft in JSON format, ensuring all required fields are present.

- **Nodes Involved:**  
  - Create proposal (OpenAI)  

- **Node Details:**  
  - **Create proposal**  
    - *Type & Role:* OpenAI LLM node generating proposal JSON from simplified inputs.  
    - *Configuration:*  
      - Model: GPT-4o-mini  
      - System prompt instructs to produce a JSON with specific fields (proposalTitle, clientName, problem, solution, milestones, pricing, etc.) with a spartan, casual, yet professional tone.  
      - User prompt dynamically fills fields from the "Set proposal values" node.  
      - JSON output enabled for downstream parsing.  
    - *Connections:* Input ← Set proposal values; Output → Rewrite proposal.  
    - *Edge Cases:*  
      - Potential incomplete or malformed JSON responses.  
      - LLM rate limits or timeouts.  
      - Input data mismatch causing empty fields.  
    - *Version Notes:* Requires OpenAI API credentials with GPT-4o-mini access.

---

#### 2.3 Proposal Rewriting and Refinement

- **Overview:**  
  This block refines the initial draft, improving clarity, tone, and flow by passing the JSON proposal through OpenRouter’s Claude 3.5 sonnet model with a structured output parser.

- **Nodes Involved:**  
  - Rewrite proposal (LangChain LLM Chain)  
  - OpenRouter Chat Model (OpenRouter LLM)  
  - Structured Output Parser (LangChain Output Parser)  

- **Node Details:**  
  - **OpenRouter Chat Model**  
    - *Type & Role:* Chat LLM node using Anthropic Claude 3.5 sonnet via OpenRouter API.  
    - *Configuration:* Model selection for Claude 3.5, no additional options.  
    - *Connections:* Input ← Rewrite proposal’s ai_languageModel input; Output → Rewrite proposal.  
    - *Edge Cases:* API auth failures, latency, unexpected model errors.

  - **Structured Output Parser**  
    - *Type & Role:* Parses the structured JSON output from the LLM, ensuring valid JSON matching a provided schema example (proposal fields).  
    - *Configuration:* Uses a JSON schema example with all proposal fields predefined.  
    - *Connections:* Input ← Rewrite proposal’s ai_outputParser; Output → Rewrite proposal.  
    - *Edge Cases:* Parsing errors if output deviates from schema, causing workflow failure.

  - **Rewrite proposal**  
    - *Type & Role:* LangChain chain node orchestrating the rewrite step.  
    - *Configuration:*  
      - Uses human message prompt template with the initial draft as input.  
      - Output parser enabled to produce structured JSON.  
      - Instructions specify a professional, helpful, friendly tone with a maximum 4-word title.  
    - *Connections:* Input ← Create proposal; Outputs → Google Drive node.  
    - *Edge Cases:* Failure in prompt rendering, parsing errors, connectivity issues with OpenRouter.

---

#### 2.4 Google Docs Template Copying

- **Overview:**  
  This block creates a new Google Docs document by copying a preformatted proposal template from Google Drive, named dynamically based on date, client company, and proposal title.

- **Nodes Involved:**  
  - Google Drive (Copy operation)  

- **Node Details:**  
  - **Google Drive**  
    - *Type & Role:* Google Drive node performing a copy operation.  
    - *Configuration:*  
      - Copies file with ID `1-NMSCV8bciEywiU5F4vxih7DCiDHu8HeC3m1LEijQ6k` from Drive ID `19v0fUzeIKVLBqBfQBJ1yK2FPCEPSA5-h`.  
      - New file name dynamically composed as: current ISO date + client company + proposal title (from Rewrite proposal node output).  
      - Copies to root folder.  
    - *Connections:* Input ← Rewrite proposal; Output → Update proposal node.  
    - *Edge Cases:*  
      - Google Drive API limits or auth errors.  
      - Template file missing or permission denied.  
      - Naming collisions or invalid characters in names.

---

#### 2.5 Variable Replacement in Google Docs

- **Overview:**  
  This block updates the newly copied Google Docs template by replacing placeholder variables with the polished proposal content from the rewriting step, finalizing the document.

- **Nodes Involved:**  
  - Update proposal (Google Docs)  

- **Node Details:**  
  - **Update proposal**  
    - *Type & Role:* Google Docs node performing text replacements.  
    - *Configuration:*  
      - Operation: Update  
      - Target document URL: taken from the copied document’s ID.  
      - Replaces all template placeholders (e.g., `{proposalTitle}`, `{clientName}`, `{problem}`, etc.) with corresponding values from the Rewrite proposal node’s output.  
      - Date uses current date formatted as yyyy-MM-dd.  
    - *Connections:* Input ← Google Drive; Output: none (workflow end).  
    - *Edge Cases:*  
      - Google Docs API rate limits or authentication failures.  
      - Placeholders missing or mismatched in template causing incomplete replacements.  
      - Large document size or formatting issues.

---

#### 2.6 Output Delivery

- **Overview:**  
  The final updated Google Docs document is ready for user access via Google Drive or Google Docs interfaces.

- **Nodes Involved:**  
  - None explicitly for delivery; final node is Update proposal.

- **Node Details:**  
  - The workflow ends after variable replacement. The output document URL or ID is present in the last node’s output for downstream use or notification (not implemented here).  
  - User can access the document in their Google Drive root folder with the generated name.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                       | Input Node(s)                  | Output Node(s)         | Sticky Note                                                                                                                                                                                                                   |
|-----------------------------|-----------------------------------------|------------------------------------|-------------------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                          | Entry point for manual workflow execution | None                          | Set proposal values      | ## Manual Input You provide the core proposal values and requirements                                                                                                                                                |
| Set proposal values          | Set                                     | Assigns initial proposal input values | When clicking ‘Execute workflow’ | Create proposal          | ## Manual Input You provide the core proposal values and requirements                                                                                                                                                |
| Create proposal             | OpenAI (GPT-4o-mini)                     | Generates detailed proposal draft JSON | Set proposal values             | Rewrite proposal         | ## Write Proposal OpenAI's LLM expands your initial inputs into a detailed, comprehensive proposal draft                                                                                                   |
| Rewrite proposal            | LangChain LLM Chain with output parser  | Refines and polishes the draft proposal | Create proposal; OpenRouter Chat Model; Structured Output Parser | Google Drive             | ## Rewrite proposal OpenRouter's Claude model refines the expanded content, improving clarity, flow, and overall readability                                                                                   |
| OpenRouter Chat Model       | OpenRouter Claude 3.5 Chat Model         | Underlying LLM for rewriting process | Rewrite proposal (ai_languageModel) | Rewrite proposal         | ## Rewrite proposal OpenRouter's Claude model refines the expanded content, improving clarity, flow, and overall readability                                                                                   |
| Structured Output Parser    | LangChain Structured JSON Output Parser | Parses Claude output to structured JSON | Rewrite proposal (ai_outputParser) | Rewrite proposal         | ## Rewrite proposal OpenRouter's Claude model refines the expanded content, improving clarity, flow, and overall readability                                                                                   |
| Google Drive                | Google Drive Copy                        | Copies proposal template to new doc | Rewrite proposal               | Update proposal          | ## Copy Google Doc from Drive Automatically copies your templated Google Doc to create a new proposal document                                                                                              |
| Update proposal             | Google Docs                             | Replaces template variables with final proposal content | Google Drive                  | None                    | ## Variable Replacement It searches through the copied document and replaces all template variables with the polished content from Claude. The formatted proposal is ready in Google Docs                         |
| Sticky Note1               | Sticky Note                            | Documentation and video walkthrough  | None                          | None                    | ## Proposal Generator Node Below is a video walkthrough of the Proposal Generator: [Watch the video](https://waveten.neetorecord.com/watch/bd8c7fc6122a3bc3df25) This automated workflow transforms basic proposal inputs into polished, professionally formatted documents in Google Docs. ### Prerequisites - OpenRouter Account - Required for accessing Claude's rewriting capabilities - Templated Google Doc - A pre-formatted document containing variables that match the output fields |
| Sticky Note2               | Sticky Note                            | Explains manual input block         | None                          | None                    | ## Manual Input You provide the core proposal values and requirements                                                                                                                                                    |
| Sticky Note3               | Sticky Note                            | Explains proposal generation block  | None                          | None                    | ## Write Proposal OpenAI's LLM expands your initial inputs into a detailed, comprehensive proposal draft                                                                                                              |
| Sticky Note4               | Sticky Note                            | Explains rewriting proposal block   | None                          | None                    | ## Rewrite proposal OpenRouter's Claude model refines the expanded content, improving clarity, flow, and overall readability                                                                                          |
| Sticky Note5               | Sticky Note                            | Explains Google Drive copy block    | None                          | None                    | ## Copy Google Doc from Drive Automatically copies your templated Google Doc to create a new proposal document                                                                                                         |
| Sticky Note6               | Sticky Note                            | Explains Google Docs replacement block | None                          | None                    | ## Variable Replacement It searches through the copied document and replaces all template variables with the polished content from Claude. The formatted proposal is ready in Google Docs                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: Entry point for manual execution. No parameters needed.

2. **Create Set Node "Set proposal values":**  
   - Assign static string values for:  
     - Name company: "Google"  
     - Name client: "Sundar Pichai"  
     - Problem: "There is no clear way of generating a proposal at scale"  
     - Solution: "Proposal generator using n8n"  
     - Timeline: "1 week"  
     - Scope: "Scope of project is only focused on clients in USA"  
     - Price: "1000000"  
   - Connect input from Manual Trigger node.

3. **Create OpenAI Node "Create proposal":**  
   - Model: GPT-4o-mini  
   - Credentials: OpenAI API with access to GPT-4o-mini  
   - Messages:  
     - System prompt: Instructions to produce a JSON proposal with specific fields (proposalTitle, clientName, clientCompany, proposalDate, problem, solution, goal, deliverables, scope, milestones, quantity, price, tax, totalPriceInclTax) with a spartan, casual, professional tone.  
     - User prompt: JSON injecting values from the previous Set node via expressions.  
     - Assistant prompt: Context about lead gen agency and detailed format example.  
   - Enable JSON output for structured data.  
   - Connect input from Set proposal values.

4. **Create LangChain Chain Node "Rewrite proposal":**  
   - Use LangChain LLM Chain node.  
   - Messages: HumanMessagePromptTemplate using the JSON output from "Create proposal" as input.  
   - Enable output parser with JSON schema example matching proposal fields.  
   - Connect input from "Create proposal" main output.

5. **Create OpenRouter Chat Model Node:**  
   - Model: Anthropic Claude 3.5 Sonnet via OpenRouter API.  
   - Credentials: OpenRouter API account.  
   - Connect input to "Rewrite proposal" ai_languageModel input.

6. **Create Structured Output Parser Node:**  
   - Paste JSON schema example with all proposal fields required.  
   - Connect input to "Rewrite proposal" ai_outputParser input.

7. **Connect "Rewrite proposal" output to Google Drive Node:**  
   - Operation: Copy  
   - Source File ID: `1-NMSCV8bciEywiU5F4vxih7DCiDHu8HeC3m1LEijQ6k` (Google Docs proposal template)  
   - Drive ID: `19v0fUzeIKVLBqBfQBJ1yK2FPCEPSA5-h`  
   - Folder ID: Root folder  
   - Dynamic new file name: `{{ new Date().toISOString().split('T')[0] }} - {{ $json.output.clientCompany }} - {{ $json.output.proposalTitle }}`  
   - Credentials: Google Drive OAuth2 account.  

8. **Create Google Docs Node "Update proposal":**  
   - Operation: Update  
   - Document URL: Use ID from Google Drive copy node output (`{{ $json.id }}`)  
   - Replace each placeholder text (e.g., `{proposalTitle}`, `{clientName}`) with corresponding values from "Rewrite proposal" output JSON fields via expressions.  
   - Replace `{proposalDate}` with current date in yyyy-MM-dd format.  
   - Credentials: Google Docs OAuth2 account.

9. **Connect Google Drive node output to Google Docs node input.**

10. **Ensure all credentials are properly configured and authorized:**  
    - OpenAI API with GPT-4o-mini access  
    - OpenRouter API with Claude 3.5 access  
    - Google Drive OAuth2 with access to the template file and destination folder  
    - Google Docs OAuth2 with permission to edit documents in Drive

11. **Test the workflow manually by triggering the Manual Trigger node.**  
    - Verify each step outputs expected structured data.  
    - Confirm the Google Docs copy is created and populated correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Video walkthrough of the Proposal Generator demonstrating the entire workflow and usage prerequisites.                                                                  | [Watch the video](https://waveten.neetorecord.com/watch/bd8c7fc6122a3bc3df25)                                   |
| Requires: OpenRouter account for Claude rewriting capabilities, and a preformatted Google Docs template with matching variables for replacement.                         | Workflow prerequisites                                                                                            |
| OpenAI GPT-4o-mini model used for initial proposal creation ensures cost-effective yet detailed output.                                                                  | Model choice rationale                                                                                            |
| Output JSON schema used in structured parser ensures consistent data format for downstream usage and template replacement.                                              | Ensures robustness in parsing AI outputs                                                                         |
| Tax rate hardcoded as 21% and included in the proposal JSON for pricing calculations.                                                                                     | Financial detail embedded in prompts                                                                              |

---

*Disclaimer:* The provided content is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal or offensive material. All processed data is legal and publicly accessible.