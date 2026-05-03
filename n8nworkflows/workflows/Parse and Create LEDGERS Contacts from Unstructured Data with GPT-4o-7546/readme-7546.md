Parse and Create LEDGERS Contacts from Unstructured Data with GPT-4o

https://n8nworkflows.xyz/workflows/parse-and-create-ledgers-contacts-from-unstructured-data-with-gpt-4o-7546


# Parse and Create LEDGERS Contacts from Unstructured Data with GPT-4o

### 1. Workflow Overview

This workflow automates the creation of contacts in the LEDGERS system from unstructured contact data updated in a Google Sheet. It leverages AI (OpenAI GPT-4o) to parse and normalize raw data entries into a structured contact format required by LEDGERS. After parsing and structuring, it batches the data to create contacts via LEDGERS API and sends success or failure email notifications accordingly.

**Target Use Cases:**  
- Organizations maintaining contact information in Google Sheets who want to automate contact creation in LEDGERS without manual data entry.  
- Use cases where contact data is unstructured or inconsistent and requires AI-driven normalization and validation.  

**Logical Blocks:**  
1.1 Input Reception and Trigger (Google Sheets Trigger)  
1.2 AI-Based Contact Data Normalization (OpenAI Chat Model + AI Agent)  
1.3 Structured JSON Parsing (Structured Output Parser)  
1.4 Form Data Loop and Iteration (SplitInBatches + NoOp nodes)  
1.5 Contact Creation in LEDGERS (LEDGERS Node + Loop)  
1.6 Success and Failure Handling (If node + Gmail notifications)  
1.7 Sticky Note Documentation (Embedded workflow description)

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Trigger

- **Overview:**  
  Watches for any updates in a specific Google Sheet row and triggers the workflow to start processing.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**  
  - **Google Sheets Trigger**  
    - Type: Trigger node that polls Google Sheets for updates every minute.  
    - Config: Monitors a specific Sheet (ID: `1m4lbSUudsxNI-thsPBZ1sjMU2yWLSQf0MrURDjBwIlg`), Sheet Name by ID `1920008147`.  
    - Inputs: None (trigger).  
    - Outputs: Triggers workflow on new or updated rows.  
    - Edge Cases: API quota limits, network errors, permission issues for Google Sheets API.  
    - Version: 1.

#### 1.2 AI-Based Contact Data Normalization

- **Overview:**  
  Takes unstructured row data and uses OpenAI GPT-4o to clean, normalize, and structure it into a contact data JSON according to specified rules.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Contact Create Smart AI (Langchain Agent)  
  - Structured Output Parser

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Language model node using OpenAI GPT-4o-mini model.  
    - Config: Model "gpt-4o-mini", no additional options.  
    - Credentials: OpenAI API key configured with free credits.  
    - Inputs: Raw Google Sheets row data.  
    - Outputs: AI-generated text response containing normalized JSON.  
    - Edge Cases: API rate limits, incomplete or malformed data, timeouts.  
    - Version: 1.2  

  - **Contact Create Smart AI**  
    - Type: Langchain Agent node executing a detailed prompt to clean and normalize JSON input based on specific rules.  
    - Config: Complex prompt defining strict rules for parsing contact fields, validation (email format, GSTIN, PAN), normalization (state/country uppercase, address fields), and error logging.  
    - Inputs: Raw Google Sheets row data forwarded from OpenAI Chat Model.  
    - Outputs: Structured JSON object with normalized contact data and error details.  
    - Edge Cases: AI misinterpretation, missing required fields (e.g., name), invalid email, unexpected input formats.  
    - Version: 2.2  

  - **Structured Output Parser**  
    - Type: Langchain structured output parser to parse AI output JSON into usable JSON object.  
    - Config: Manual schema defining all expected fields like name, mobile_country_code, email, gstin, billing and shipping addresses, and error object.  
    - Inputs: AI text response from Contact Create Smart AI node.  
    - Outputs: Parsed JSON object with the contact data fields.  
    - Edge Cases: Parsing failures if AI output deviates from schema or contains invalid JSON.  
    - Version: 1.3  

#### 1.3 Form Data Loop and Iteration

- **Overview:**  
  Splits the structured contact data into batches (though typically processing one contact at a time), iterates through each batch to handle data processing stepwise.

- **Nodes Involved:**  
  - Form Loop (SplitInBatches)  
  - Form Iteration (NoOp)

- **Node Details:**  
  - **Form Loop**  
    - Type: SplitInBatches node to process data in batches.  
    - Config: Default batch options (likely batch size = 1 as no explicit size given).  
    - Inputs: Structured JSON contact data from AI output.  
    - Outputs: Each batch passed to the next nodes for contact creation.  
    - Edge Cases: Empty batches, batch size misconfiguration.  
    - Version: 3  

  - **Form Iteration**  
    - Type: NoOp node used as a loop iteration checkpoint.  
    - Config: No parameters; used to connect back to Form Loop for iteration continuation.  
    - Inputs: From Create a Contact node or Form Loop.  
    - Outputs: Loops back to Form Loop to continue processing remaining batches.  
    - Edge Cases: None functional; mainly structural.  
    - Version: 1  

#### 1.4 Contact Creation in LEDGERS

- **Overview:**  
  Uses the LEDGERS node to create contacts in the LEDGERS system with the structured data from AI. Includes looping for handling multiple contact creations if needed.

- **Nodes Involved:**  
  - Create a contact (LEDGERS node)  
  - LEDGERS Loop (SplitInBatches)  
  - LEDGERS Iteration (NoOp)

- **Node Details:**  
  - **Create a contact**  
    - Type: LEDGERS API integration node for creating contacts.  
    - Config: Maps all fields from AI output JSON to LEDGERS contact fields including name, pan, email, gstin, mobile, business name, billing and shipping addresses, and country codes.  
    - Inputs: Batches of structured contact data from Form Loop.  
    - Outputs: API response indicating success or error for contact creation.  
    - Edge Cases: API authorization errors, malformed data, missing required fields, duplicate contact errors.  
    - Version: 1  

  - **LEDGERS Loop**  
    - Type: SplitInBatches node to process LEDGERS contact creation responses in batches.  
    - Config: Default batching options.  
    - Inputs: From Create a contact node.  
    - Outputs: Passes responses to Success/Failure node or loops again as needed.  
    - Edge Cases: Empty batches, API response handling errors.  
    - Version: 3  

  - **LEDGERS Iteration**  
    - Type: NoOp node used for iterating through LEDGERS Loop batches.  
    - Config: No parameters; loops back to LEDGERS Loop.  
    - Inputs: From Success/Failure node or LEDGERS Loop.  
    - Outputs: Loops back to LEDGERS Loop.  
    - Edge Cases: None functional; structural only.  
    - Version: 1  

#### 1.5 Success and Failure Handling

- **Overview:**  
  Evaluates whether contact creation succeeded or failed and sends corresponding email notifications via Gmail.

- **Nodes Involved:**  
  - Success/Failure (If node)  
  - Contact Success (Gmail)  
  - Contact Failed (Gmail)

- **Node Details:**  
  - **Success/Failure**  
    - Type: If node evaluating if errorMessage exists in LEDGERS response JSON.  
    - Config: Condition checks if `errorMessage` key does not exist (success) or exists (failure).  
    - Inputs: Output from LEDGERS Loop batches.  
    - Outputs: Routes to Contact Success node on success, Contact Failed node on failure.  
    - Edge Cases: Undefined or unexpected response formats, missing errorMessage key.  
    - Version: 2.2 (inactive in current workflow but included)  

  - **Contact Success**  
    - Type: Gmail node sending email notification on success.  
    - Config: HTML email with success message, includes JSON response data.  
    - Inputs: From Success/Failure node success branch.  
    - Outputs: End node.  
    - Edge Cases: Gmail API quota, authentication errors, email formatting issues.  
    - Version: 2.1  

  - **Contact Failed**  
    - Type: Gmail node sending email notification on failure.  
    - Config: HTML email with failure message including error details from JSON.  
    - Inputs: From Success/Failure node failure branch.  
    - Outputs: End node.  
    - Edge Cases: Same as Contact Success.  
    - Version: 2.1  

#### 1.6 Sticky Note Documentation

- **Overview:**  
  Provides embedded workflow documentation summarizing the flow, purpose, and use cases for users and maintainers.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - Type: Visual sticky note node for workflow annotation.  
    - Content: Detailed summary of workflow logic, node roles, and use case description.  
    - Position: Top-left corner visually.  
    - Inputs/Outputs: None.  
    - Version: 1  

---

### 3. Summary Table

| Node Name              | Node Type                                  | Functional Role                               | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                   |
|------------------------|--------------------------------------------|-----------------------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| Google Sheets Trigger   | n8n-nodes-base.googleSheetsTrigger         | Trigger workflow on Google Sheet row update  | None                        | Contact Create Smart AI        | This n8n workflow automatically creates contacts in LEDGERS using AI, based on data updates in a Google Sheet. |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Generate normalized contact JSON from raw data | Google Sheets Trigger       | Contact Create Smart AI        |                                                                                               |
| Contact Create Smart AI | @n8n/n8n-nodes-langchain.agent             | Parse and normalize contact JSON with AI     | Google Sheets Trigger, OpenAI Chat Model (ai_languageModel) | Form Loop                     |                                                                                               |
| Structured Output Parser| @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI output into structured JSON          | Contact Create Smart AI      | Contact Create Smart AI        |                                                                                               |
| Form Loop              | n8n-nodes-base.splitInBatches               | Batch processing of contact data              | Contact Create Smart AI      | Create a contact, Form Iteration |                                                                                               |
| Form Iteration         | n8n-nodes-base.noOp                          | Loop iteration over batches                    | Create a contact             | Form Loop                     |                                                                                               |
| Create a contact       | @ledgers/n8n-nodes-ledgers-cloud.ledgers    | Create contact in LEDGERS system               | Form Loop                   | LEDGERS Loop                  |                                                                                               |
| LEDGERS Loop           | n8n-nodes-base.splitInBatches               | Batch processing of LEDGERS API responses     | Create a contact            | Success/Failure, LEDGERS Iteration |                                                                                               |
| LEDGERS Iteration      | n8n-nodes-base.noOp                          | Loop iteration for LEDGERS batches             | Success/Failure, LEDGERS Loop | LEDGERS Loop                  |                                                                                               |
| Success/Failure        | n8n-nodes-base.if                            | Check if contact creation succeeded or failed | LEDGERS Loop                | Contact Success, Contact Failed |                                                                                               |
| Contact Success        | n8n-nodes-base.gmail                         | Send success email notification                | Success/Failure             | None                         |                                                                                               |
| Contact Failed         | n8n-nodes-base.gmail                         | Send failure email notification                | Success/Failure             | None                         |                                                                                               |
| Sticky Note            | n8n-nodes-base.stickyNote                    | Workflow documentation and summary             | None                        | None                         | This n8n workflow automatically creates contacts in LEDGERS using AI, based on data updates in a Google Sheet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Poll every minute for changes  
   - Configure with Google Sheets credentials  
   - Set Document ID: `1m4lbSUudsxNI-thsPBZ1sjMU2yWLSQf0MrURDjBwIlg`  
   - Set Sheet Name/ID: `1920008147`  

2. **Add OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Connect input from Google Sheets Trigger node  
   - Configure OpenAI API credentials  

3. **Add Contact Create Smart AI Node**  
   - Type: Langchain Agent  
   - Prompt: Use detailed JSON cleaning and normalization prompt (as per node configuration)  
   - Connect input from Google Sheets Trigger (main) and OpenAI Chat Model (ai_languageModel output)  
   - Enable output parser: true  

4. **Add Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - Manual schema input as specified, including all contact fields and errors object  
   - Connect ai_outputParser input from Contact Create Smart AI  

5. **Add Form Loop Node (SplitInBatches)**  
   - Type: SplitInBatches  
   - Default batch size (or set batch size = 1)  
   - Connect input from Contact Create Smart AI main output  

6. **Add Form Iteration Node (NoOp)**  
   - Type: NoOp  
   - Connect output from Create a contact node to this node  
   - Connect output back to Form Loop for looping  

7. **Add Create a contact Node (LEDGERS Integration)**  
   - Type: LEDGERS contact creation node  
   - Map fields from the structured JSON output (`$json['output']`), including nested billing and shipping addresses, mobile country code, pan, gstin, email, mobile, business name  
   - Connect input from Form Loop output  
   - Connect output to LEDGERS Loop node  

8. **Add LEDGERS Loop Node (SplitInBatches)**  
   - Type: SplitInBatches  
   - Default batch options  
   - Connect input from Create a contact output  
   - Connect outputs to Success/Failure node and LEDGERS Iteration node  

9. **Add LEDGERS Iteration Node (NoOp)**  
   - Type: NoOp  
   - Connect outputs from Success/Failure and LEDGERS Loop nodes  
   - Connect output back to LEDGERS Loop for iteration  

10. **Add Success/Failure Node (If)**  
    - Type: If node  
    - Condition: Check if `$json['errorMessage']` does not exist (success) or exists (failure)  
    - Connect input from LEDGERS Loop output  
    - Connect success output to Contact Success, failure output to Contact Failed  

11. **Add Contact Success Node (Gmail)**  
    - Type: Gmail  
    - Configure Gmail OAuth2 credentials  
    - Email subject: "Regarding Create Contact via LEDGERS N8N"  
    - Email body: HTML template with success message and JSON response data  
    - Connect input from Success/Failure success branch  

12. **Add Contact Failed Node (Gmail)**  
    - Type: Gmail  
    - Configure Gmail OAuth2 credentials  
    - Email subject: "Regarding Create Contact via LEDGERS"  
    - Email body: HTML template with failure message and error JSON details  
    - Connect input from Success/Failure failure branch  

13. **Add Sticky Note**  
    - Add a sticky note node anywhere convenient (recommended top-left)  
    - Paste the workflow summary and description for documentation  

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates contact creation from unstructured Google Sheets data using AI and integrates with LEDGERS API. | Workflow purpose summary (also documented in Sticky Note node).                                |
| OpenAI GPT-4o-mini is used for AI parsing and normalization of contact data.                                             | Requires OpenAI API credentials with proper quota.                                            |
| Ensure Google Sheets API and Gmail API credentials have necessary permissions and OAuth scopes configured in n8n.       | Credential configuration requirement for Google Sheets and Gmail nodes.                        |
| The detailed AI prompt enforces strict rules to avoid placeholder data and only use available input data.               | Important for data quality and avoiding AI hallucination.                                     |
| The workflow’s Success/Failure node is currently inactive but included for notification on contact creation results.    | Can be activated to enable email notifications on contact creation success or failure.         |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies with all current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.