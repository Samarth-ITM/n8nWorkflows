Clone LinkedIn Writing Styles with AI Analysis & Prompt Generation to Airtable

https://n8nworkflows.xyz/workflows/clone-linkedin-writing-styles-with-ai-analysis---prompt-generation-to-airtable-4650


# Clone LinkedIn Writing Styles with AI Analysis & Prompt Generation to Airtable

### 1. Workflow Overview

This workflow automates the process of cloning LinkedIn writing styles by scraping LinkedIn posts, analyzing their writing style using AI language models, generating style-specific prompts, and storing the results in Airtable. It is designed for users who want to study or replicate effective LinkedIn content styles, leveraging AI-powered natural language processing and prompt engineering.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Starts the workflow via a form trigger.
- **1.2 Data Collection:** Fetches LinkedIn posts asynchronously with scraping status checks.
- **1.3 Data Processing & Aggregation:** Combines collected posts into a single dataset.
- **1.4 AI Analysis:** Uses AI models to analyze writing style and generate custom prompts.
- **1.5 Data Storage:** Saves the AI-generated insights and prompts into Airtable for further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives the initiation trigger from a user form submission to start the LinkedIn analysis process.

**Nodes Involved:**  
- 0) Start LinkedIn Analysis

**Node Details:**  
- **0) Start LinkedIn Analysis**  
  - Type: Form Trigger  
  - Role: Entry point node that listens for form submissions to launch the workflow.  
  - Configuration: Standard form trigger with a webhook ID. No preset parameters are specified.  
  - Inputs: None (trigger node)  
  - Outputs: Connected downstream to "1) Fetch LinkedIn Posts"  
  - Edge Cases: Form submission failures or webhook misconfiguration could prevent workflow start.  
  - Notes: Ensures manual or automated external initiation.

---

#### 1.2 Data Collection

**Overview:**  
Fetches LinkedIn posts asynchronously by triggering a scraping process, waiting, checking the scraping status repeatedly until completion, then retrieving the full data.

**Nodes Involved:**  
- 1) Fetch LinkedIn Posts  
- 2) Wait for Scraping  
- 3) Check Scraping Status  
- 4) Verify Completion  
- 5) Get LinkedIn Data

**Node Details:**  

- **1) Fetch LinkedIn Posts**  
  - Type: HTTP Request  
  - Role: Initiates scraping of LinkedIn posts, likely calling an external API or service.  
  - Configuration: HTTP request details are abstracted but expected to trigger scraping.  
  - Inputs: From "0) Start LinkedIn Analysis"  
  - Outputs: To "2) Wait for Scraping"  
  - Edge Cases: HTTP errors, invalid API endpoint, or authentication failures.

- **2) Wait for Scraping**  
  - Type: Wait Node  
  - Role: Pauses workflow execution to allow scraping to complete.  
  - Configuration: Uses webhook ID for continuation; likely waits predefined time or event.  
  - Inputs: From "1) Fetch LinkedIn Posts" and also from "4) Verify Completion" on incomplete status.  
  - Outputs: To "3) Check Scraping Status"  
  - Edge Cases: Timeout if scraping takes too long; webhook failure or lost triggers.

- **3) Check Scraping Status**  
  - Type: HTTP Request  
  - Role: Polls scraping status API to verify if data collection is complete.  
  - Configuration: Calls status endpoint; no direct parameters shown but expected to interpret response.  
  - Inputs: From "2) Wait for Scraping"  
  - Outputs: To "4) Verify Completion"  
  - Edge Cases: API downtime, malformed responses, or misinterpretation of status.

- **4) Verify Completion**  
  - Type: If Node  
  - Role: Logical condition to branch workflow based on scraping completion status.  
  - Configuration: Evaluates status from "3) Check Scraping Status."  
  - Inputs: From "3) Check Scraping Status"  
  - Outputs:  
    - True branch to "5) Get LinkedIn Data" (scraping done)  
    - False branch back to "2) Wait for Scraping" (scraping ongoing)  
  - Edge Cases: Incorrect or ambiguous status could cause infinite loops or premature continuation.

- **5) Get LinkedIn Data**  
  - Type: HTTP Request  
  - Role: Retrieves the fully scraped LinkedIn posts after completion confirmation.  
  - Configuration: Calls data retrieval endpoint, expected to return post data.  
  - Inputs: From "4) Verify Completion" (true branch)  
  - Outputs: To "6) Combine Posts"  
  - Edge Cases: Data retrieval failure, incomplete data, or API errors.

---

#### 1.3 Data Processing & Aggregation

**Overview:**  
Aggregates multiple LinkedIn posts into a combined dataset for subsequent AI analysis.

**Nodes Involved:**  
- 6) Combine Posts

**Node Details:**  
- **6) Combine Posts**  
  - Type: Aggregate  
  - Role: Concatenates or aggregates all retrieved post data into a unified format.  
  - Configuration: Uses aggregation functions to merge post contents, likely concatenating text fields.  
  - Inputs: From "5) Get LinkedIn Data"  
  - Outputs: To "7) Analyze Writing Style"  
  - Edge Cases: Empty datasets or malformed post content could affect aggregation.

---

#### 1.4 AI Analysis

**Overview:**  
Analyzes the combined LinkedIn posts using AI language models to extract writing style features and generate style-based prompts.

**Nodes Involved:**  
- 7) Analyze Writing Style  
- 8) Generate Style Prompt  
- Style Analysis Model  
- Prompt Creation Model

**Node Details:**  

- **7) Analyze Writing Style**  
  - Type: LangChain Chain LLM  
  - Role: Executes an AI chain to analyze writing style characteristics from combined posts.  
  - Configuration: Connected to "Style Analysis Model" for AI processing.  
  - Inputs: From "6) Combine Posts" (main) and "Style Analysis Model" (ai_languageModel)  
  - Outputs: To "8) Generate Style Prompt"  
  - Edge Cases: AI response delays, model errors, or unexpected output formats.

- **8) Generate Style Prompt**  
  - Type: LangChain Chain LLM  
  - Role: Uses AI to generate a prompt that replicates or leverages the analyzed writing style.  
  - Configuration: Connected to "Prompt Creation Model" for AI generation.  
  - Inputs: From "7) Analyze Writing Style" (main) and "Prompt Creation Model" (ai_languageModel)  
  - Outputs: To "9) Save to Airtable"  
  - Edge Cases: Model inconsistencies, token limits, or generation failures.

- **Style Analysis Model**  
  - Type: LangChain LM Chat Open Router  
  - Role: AI language model endpoint for analyzing writing style.  
  - Configuration: Configured with OpenAI or compatible credentials; handles chat-based prompt processing.  
  - Inputs: Connected as ai_languageModel input to "7) Analyze Writing Style"  
  - Outputs: To "7) Analyze Writing Style"  
  - Edge Cases: API rate limits, authentication errors, or network issues.

- **Prompt Creation Model**  
  - Type: LangChain LM Chat Open Router  
  - Role: AI language model endpoint for generating style prompts.  
  - Configuration: Similar to Style Analysis Model but used for prompt creation.  
  - Inputs: Connected as ai_languageModel input to "8) Generate Style Prompt"  
  - Outputs: To "8) Generate Style Prompt"  
  - Edge Cases: Same as Style Analysis Model.

---

#### 1.5 Data Storage

**Overview:**  
Stores the analyzed writing style and generated prompt into an Airtable base for record-keeping and further action.

**Nodes Involved:**  
- 9) Save to Airtable

**Node Details:**  
- **9) Save to Airtable**  
  - Type: Airtable  
  - Role: Inserts or updates records in Airtable with AI-generated analysis and prompts.  
  - Configuration: Airtable credentials required; table and field mappings configured to store style data.  
  - Inputs: From "8) Generate Style Prompt"  
  - Outputs: None (terminal node)  
  - Edge Cases: Airtable API limits, credential expiration, or incorrect field mapping.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                      | Input Node(s)           | Output Node(s)            | Sticky Note |
|-------------------------|--------------------------------|------------------------------------|------------------------|---------------------------|-------------|
| Workflow Instructions    | Sticky Note                    | Documentation placeholder           |                        |                           |             |
| Data Collection Note     | Sticky Note                    | Documentation placeholder           |                        |                           |             |
| Analysis Note            | Sticky Note                    | Documentation placeholder           |                        |                           |             |
| Storage Note             | Sticky Note                    | Documentation placeholder           |                        |                           |             |
| 0) Start LinkedIn Analysis | Form Trigger                  | Workflow entry point                |                        | 1) Fetch LinkedIn Posts   |             |
| 1) Fetch LinkedIn Posts  | HTTP Request                   | Initiates LinkedIn post scraping    | 0) Start LinkedIn Analysis | 2) Wait for Scraping     |             |
| 2) Wait for Scraping     | Wait                          | Pauses workflow for scraping delay | 1) Fetch LinkedIn Posts; 4) Verify Completion (false branch) | 3) Check Scraping Status |             |
| 3) Check Scraping Status | HTTP Request                   | Polls scraping completion status    | 2) Wait for Scraping    | 4) Verify Completion      |             |
| 4) Verify Completion     | If                            | Branches on scraping completion     | 3) Check Scraping Status | 5) Get LinkedIn Data (true); 2) Wait for Scraping (false) |             |
| 5) Get LinkedIn Data     | HTTP Request                   | Retrieves scraped post data          | 4) Verify Completion (true) | 6) Combine Posts        |             |
| 6) Combine Posts         | Aggregate                     | Aggregates post data                 | 5) Get LinkedIn Data    | 7) Analyze Writing Style  |             |
| 7) Analyze Writing Style | LangChain Chain LLM           | Analyzes writing style with AI      | 6) Combine Posts; Style Analysis Model (ai_languageModel) | 8) Generate Style Prompt |             |
| 8) Generate Style Prompt | LangChain Chain LLM           | Generates prompt based on style     | 7) Analyze Writing Style; Prompt Creation Model (ai_languageModel) | 9) Save to Airtable |             |
| Style Analysis Model     | LangChain LM Chat Open Router | AI model for style analysis         |                        | 7) Analyze Writing Style  |             |
| Prompt Creation Model    | LangChain LM Chat Open Router | AI model for prompt generation      |                        | 8) Generate Style Prompt  |             |
| 9) Save to Airtable      | Airtable                      | Saves analysis and prompt to Airtable | 8) Generate Style Prompt |                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "0) Start LinkedIn Analysis"  
   - Set it to listen for form submissions with a unique webhook ID.

2. **Add an HTTP Request node** named "1) Fetch LinkedIn Posts"  
   - Configure it to call the LinkedIn scraping API endpoint to start post fetching.  
   - Connect output from "0) Start LinkedIn Analysis" to this node.

3. **Add a Wait node** named "2) Wait for Scraping"  
   - Configure it to pause for a defined period or wait for a webhook signal.  
   - Connect output from "1) Fetch LinkedIn Posts" to this node.

4. **Add an HTTP Request node** named "3) Check Scraping Status"  
   - Configure it to poll the scraping status endpoint.  
   - Connect output from "2) Wait for Scraping" to this node.

5. **Add an If node** named "4) Verify Completion"  
   - Configure the condition to check if scraping status equals "completed" (or equivalent).  
   - Connect output from "3) Check Scraping Status" to this node.

6. **Configure the If node branches:**  
   - **True branch:** Add an HTTP Request node named "5) Get LinkedIn Data" to fetch the full posts data. Connect from the true output.  
   - **False branch:** Loop back to "2) Wait for Scraping" to continue waiting.

7. **Add an Aggregate node** named "6) Combine Posts"  
   - Configure to merge post contents into a single textual dataset.  
   - Connect output from "5) Get LinkedIn Data" to this node.

8. **Add a LangChain Chain LLM node** named "7) Analyze Writing Style"  
   - Connect the main input from "6) Combine Posts."  
   - Add a LangChain LM Chat Open Router node named "Style Analysis Model" configured for your OpenAI or compatible AI credentials.  
   - Link "Style Analysis Model" as the AI language model input to this node.

9. **Add a LangChain Chain LLM node** named "8) Generate Style Prompt"  
   - Connect main input from "7) Analyze Writing Style."  
   - Add a LangChain LM Chat Open Router node named "Prompt Creation Model," also configured with AI credentials.  
   - Link "Prompt Creation Model" as the AI language model input.

10. **Add an Airtable node** named "9) Save to Airtable"  
    - Configure Airtable credentials and select the target base and table.  
    - Map fields to save the analyzed writing style data and generated prompt.  
    - Connect output from "8) Generate Style Prompt" to this node.

11. **Connect all nodes following the sequence and branching logic as per the summary table.**

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                |
|----------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow is designed to clone LinkedIn writing styles by combining scraping and AI analysis. | Workflow purpose description                     |
| Requires OpenAI or compatible AI credentials for LangChain nodes (Style Analysis and Prompt Creation Models). | AI integration requirements                      |
| Airtable credentials must be configured for data storage node.                              | Airtable integration setup                       |
| Uses asynchronous polling pattern to handle LinkedIn scraping process with wait and status check nodes. | Scraping and polling design pattern              |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the "Clone LinkedIn Writing Styles with AI Analysis & Prompt Generation to Airtable" n8n workflow.