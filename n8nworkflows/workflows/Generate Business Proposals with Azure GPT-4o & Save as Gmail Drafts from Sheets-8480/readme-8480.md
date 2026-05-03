Generate Business Proposals with Azure GPT-4o & Save as Gmail Drafts from Sheets

https://n8nworkflows.xyz/workflows/generate-business-proposals-with-azure-gpt-4o---save-as-gmail-drafts-from-sheets-8480


# Generate Business Proposals with Azure GPT-4o & Save as Gmail Drafts from Sheets

### 1. Workflow Overview

This workflow automates the generation of business proposals by leveraging Azure GPT-4o language model via LangChain integration and saves the generated proposals as Gmail drafts. Data rows from Google Sheets serve as input, with each row representing a potential proposal. The workflow includes logic to check proposal status, process valid entries through AI generation, format results, and finally create draft emails in Gmail for review and sending.

Logical blocks:

- **1.1 Input Reception:** Trigger and data retrieval from Google Sheets  
- **1.2 Proposal Status Check:** Filtering rows based on proposal status  
- **1.3 Batch Processing:** Handling each valid row individually  
- **1.4 AI Processing:** Using Azure OpenAI GPT-4o through LangChain agent for proposal generation  
- **1.5 Post-Processing:** Formatting AI output via a code node  
- **1.6 Draft Creation:** Creating Gmail draft emails with the generated proposal content  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow manually and retrieves relevant rows from a Google Sheet containing business proposal data.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get row(s) in sheet

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on-demand  
  - *Config:* No parameters; triggers workflow execution manually  
  - *Input:* None  
  - *Output:* Initiates flow to Google Sheets node  
  - *Failure Types:* N/A (manual trigger)  

- **Get row(s) in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Reads rows from the configured Google Sheet  
  - *Config:* Expected to be configured with spreadsheet ID, sheet name, and range or filter criteria to fetch relevant rows  
  - *Expressions:* Likely uses expressions to specify dynamic ranges or filters based on use case  
  - *Input:* Trigger node output  
  - *Output:* Rows array to next node (proposal status check)  
  - *Failure Types:* Authentication errors, API rate limits, empty or invalid sheet range, network errors  

---

#### 2.2 Proposal Status Check

**Overview:**  
Filters input rows to process only those proposals that meet specific criteria (e.g., status not already processed).

**Nodes Involved:**  
- checks propsal status (If node)

**Node Details:**  

- **checks propsal status**  
  - *Type:* If node  
  - *Role:* Conditionally filters rows based on proposal status field  
  - *Config:* Checks if a given row’s proposal status field meets criteria (e.g., "Pending", "New")  
  - *Expressions:* Uses expression to compare row data fields to desired status values  
  - *Input:* Rows from Google Sheets  
  - *Output:* Passes rows meeting criteria to batch processing; others ignored or routed elsewhere  
  - *Failure Types:* Expression evaluation errors if fields are missing or malformed  

---

#### 2.3 Batch Processing

**Overview:**  
Processes each filtered row individually by splitting the array into batches of one item, enabling sequential AI generation.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)

**Node Details:**  

- **Loop Over Items**  
  - *Type:* SplitInBatches node  
  - *Role:* Splits rows array into single-item batches for sequential processing  
  - *Config:* Batch size set to 1 (default or explicit)  
  - *Input:* Filtered rows from If node  
  - *Output:* Single row batch forwarded to AI processing  
  - *Failure Types:* If input is empty, node outputs no batches; handles gracefully  

---

#### 2.4 AI Processing

**Overview:**  
Uses LangChain’s AI Agent with Azure OpenAI GPT-4o to generate a business proposal based on the input row’s data.

**Nodes Involved:**  
- AI Agent (LangChain Agent node)  
- Azure OpenAI Chat Model (LangChain LLM Model node)

**Node Details:**  

- **Azure OpenAI Chat Model**  
  - *Type:* LangChain Azure OpenAI Chat Model node  
  - *Role:* Provides GPT-4o model interface for text generation  
  - *Config:* Azure credentials, model parameters (temperature, max tokens, deployment name)  
  - *Input:* Messages or prompts from AI Agent node via ai_languageModel connection  
  - *Output:* Model completions to AI Agent  
  - *Failure Types:* API key/authentication errors, rate limits, timeouts  
  - *Version:* Requires LangChain nodes v1+  

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Orchestrates prompt creation and calls LLM to generate proposal text  
  - *Config:* Agent type (e.g., “zero-shot” or custom), prompt templates referencing input fields from batch item  
  - *Input:* Single batch item from SplitInBatches node, LLM response from Azure OpenAI Chat Model  
  - *Output:* Generated proposal text to Code node  
  - *Failure Types:* Prompt logic errors, missing data fields, LLM call failures  

---

#### 2.5 Post-Processing

**Overview:**  
Formats and prepares the AI-generated proposal text for email draft creation.

**Nodes Involved:**  
- Code node

**Node Details:**  

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Processes AI output to format email subject, body, recipients, or other metadata  
  - *Config:* Custom JS code reading AI Agent output, extracting relevant fields, possibly sanitizing or structuring email content  
  - *Input:* AI Agent output  
  - *Output:* Structured data for Gmail draft creation  
  - *Failure Types:* Runtime errors, undefined properties, unexpected AI output formats  

---

#### 2.6 Draft Creation

**Overview:**  
Creates Gmail draft emails from the prepared proposal content for manual review and sending.

**Nodes Involved:**  
- Create a draft (Gmail node)

**Node Details:**  

- **Create a draft**  
  - *Type:* Gmail node  
  - *Role:* Creates a draft email in Gmail account  
  - *Config:* Uses OAuth2 credentials for Gmail, sets draft recipient(s), subject, and body from Code node output  
  - *Input:* Formatted proposal data  
  - *Output:* Draft creation confirmation or error  
  - *Failure Types:* Authentication errors, permission issues, invalid email addresses, API limits  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                 | Input Node(s)                    | Output Node(s)            | Sticky Note                   |
|----------------------------|----------------------------------|--------------------------------|---------------------------------|---------------------------|------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow start trigger          | None                            | Get row(s) in sheet       |                              |
| Get row(s) in sheet        | Google Sheets                    | Retrieve proposal input data    | When clicking ‘Execute workflow’ | checks propsal status      |                              |
| checks propsal status      | If                              | Filter rows by proposal status  | Get row(s) in sheet             | Loop Over Items           |                              |
| Loop Over Items            | SplitInBatches                   | Batch single rows for processing| checks propsal status           | AI Agent (on false branch) |                              |
| AI Agent                  | LangChain Agent                   | Generate proposal using GPT-4o  | Loop Over Items (single batch)  | Code                      |                              |
| Azure OpenAI Chat Model    | LangChain LLM Model (Azure GPT)  | GPT-4o model interface          | AI Agent (ai_languageModel)     | AI Agent                  |                              |
| Code                      | Code                            | Format AI output for Gmail draft| AI Agent                       | Create a draft            |                              |
| Create a draft            | Gmail                           | Save proposal as Gmail draft    | Code                           | None                      |                              |
| Sticky Note               | Sticky Note                     | Comments or instructions        | None                           | None                      | Multiple sticky notes present but empty content |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed. This node triggers the workflow manually.

2. **Create Google Sheets node**  
   - Name: `Get row(s) in sheet`  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Parameters: Set Spreadsheet ID, Sheet Name, and range or filter to fetch proposal rows.  
   - Connect output of Manual Trigger to this node.

3. **Create If node**  
   - Name: `checks propsal status`  
   - Parameters: Set condition to check if the proposal status column (e.g., "Status") equals "Pending" or other criteria indicating the proposal needs processing.  
   - Connect output of Google Sheets node to input of this If node.

4. **Create SplitInBatches node**  
   - Name: `Loop Over Items`  
   - Parameters: Batch size = 1 (process one row at a time).  
   - Connect the "true" output of the If node to this node.

5. **Create LangChain Azure OpenAI Chat Model node**  
   - Name: `Azure OpenAI Chat Model`  
   - Credentials: Configure Azure OpenAI API credentials (subscription key, endpoint).  
   - Parameters: Set deployment/model name (e.g., GPT-4o), temperature, max tokens, and other relevant settings.  
   - This node will serve as the language model for the AI Agent.

6. **Create LangChain Agent node**  
   - Name: `AI Agent`  
   - Parameters: Define prompt templates to generate proposals based on input data fields per item.  
   - Connect input from SplitInBatches node.  
   - Link the `ai_languageModel` input of this node to the output of the Azure OpenAI Chat Model node.

7. **Create Code node**  
   - Name: `Code`  
   - Parameters: Write JavaScript code to extract the proposal text from AI Agent output, structure email subject, body, and recipient fields.  
   - Connect output of AI Agent node to this node.

8. **Create Gmail node**  
   - Name: `Create a draft`  
   - Credentials: Set up Gmail OAuth2 credentials with draft creation scope.  
   - Parameters: Use data from Code node to set recipient(s), subject, and email body content.  
   - Connect output of Code node to this node.

9. **Connect the nodes as follows:**  
   - Manual Trigger → Google Sheets → If node → SplitInBatches → AI Agent → Code → Gmail draft creation.  
   - Connect Azure OpenAI Chat Model to AI Agent's `ai_languageModel` input.

10. **Test the workflow:**  
    - Execute manually and verify that rows with pending status generate proposals, which are then saved as Gmail drafts.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                           |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses LangChain nodes for advanced AI model orchestration with Azure OpenAI GPT-4o. | LangChain integration documentation in n8n official docs |
| Gmail draft creation requires OAuth2 credentials with appropriate scopes for draft management.  | Gmail API OAuth scopes: https://developers.google.com/gmail/api/auth/scopes |
| Google Sheets node requires OAuth2 credentials and correct spreadsheet access permissions.      | Google Sheets API: https://developers.google.com/sheets/api |
| The workflow is designed for batch processing proposals but processes one row at a time to avoid API rate limits. | Best practice for external API usage                       |
| Empty sticky notes are present in the workflow but contain no comments or instructions.          | No additional contextual notes                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.