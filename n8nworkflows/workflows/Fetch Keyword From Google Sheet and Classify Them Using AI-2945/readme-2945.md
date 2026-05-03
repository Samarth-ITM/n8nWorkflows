Fetch Keyword From Google Sheet and Classify Them Using AI

https://n8nworkflows.xyz/workflows/fetch-keyword-from-google-sheet-and-classify-them-using-ai-2945


# Fetch Keyword From Google Sheet and Classify Them Using AI

### 1. Workflow Overview

This workflow automates the classification of keywords from a Google Sheet to identify if they reference known IT software, services, tools, or apps. It is designed primarily for marketers, SEO specialists, and content managers who want to categorize large keyword lists for targeted SEO campaigns or market analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetch keywords from a specified Google Sheet.
- **1.3 Batch Processing:** Split keywords into manageable batches to avoid API rate limits.
- **1.4 Rate Limiting Control:** Introduce wait times to prevent exceeding API call limits.
- **1.5 AI Classification:** Use an AI agent powered by OpenAI to analyze each keyword and classify it as referencing an IT service or not.
- **1.6 Output Parsing:** Parse AI responses into structured JSON.
- **1.7 Data Update:** Write the classification results back into the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Triggers "Fetch Keywords from Sheet" node.  
    - Edge Cases: None, but workflow will not run unless manually triggered.

#### 2.2 Data Retrieval

- **Overview:** Fetches all keyword data from a specified Google Sheet document and sheet tab.
- **Nodes Involved:**  
  - Fetch Keywords from Sheet
- **Node Details:**

  - **Fetch Keywords from Sheet**  
    - Type: Google Sheets  
    - Role: Reads keyword data from a Google Sheet document.  
    - Configuration:  
      - Document ID: `1jzDvszQoVDV-jrAunCXqTVsiDxXVLMGqQ1zGXwfy5eU` (Google Sheet URL provided)  
      - Sheet Name: `Copy of Sheet1 1` (sheet tab ID: 1319606837)  
      - Credentials: Google Sheets OAuth2 account connected.  
    - Inputs: Trigger from manual node.  
    - Outputs: Passes full keyword dataset to "Process Keywords in Batches".  
    - Edge Cases:  
      - Authentication failure if credentials expire or are revoked.  
      - Empty or malformed sheet data.  
      - Network or API errors.  
    - Version: 4.5

#### 2.3 Batch Processing

- **Overview:** Splits the full keyword list into batches of 6 to avoid hitting API rate limits.
- **Nodes Involved:**  
  - Process Keywords in Batches
- **Node Details:**

  - **Process Keywords in Batches**  
    - Type: SplitInBatches  
    - Role: Divides input data into smaller chunks for sequential processing.  
    - Configuration: Batch size set to 6.  
    - Inputs: Keyword data from Google Sheets node.  
    - Outputs: Sends each batch downstream for processing.  
    - Edge Cases:  
      - Batch size too large may cause API throttling.  
      - Batch size too small may increase total processing time.  
    - Version: 3

#### 2.4 Rate Limiting Control

- **Overview:** Introduces a wait period between batches to prevent API rate limiting.
- **Nodes Involved:**  
  - Prevent API Rate Limiting
- **Node Details:**

  - **Prevent API Rate Limiting**  
    - Type: Wait  
    - Role: Pauses workflow execution briefly between batches.  
    - Configuration: Default wait time (not explicitly set, so defaults apply).  
    - Inputs: Triggered after the first batch is processed.  
    - Outputs: Passes control to AI Agent node.  
    - Edge Cases:  
      - Insufficient wait time may still cause rate limiting errors.  
      - Excessive wait time increases total workflow duration.  
    - Version: 1.1

#### 2.5 AI Classification

- **Overview:** Uses an AI agent to analyze each keyword and classify if it references known IT software/services.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser
- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Coordinates AI prompt and response parsing.  
    - Configuration:  
      - Input text: `keyword: {{ $json.Keyword }}` (injects current keyword)  
      - System message prompt: "Check the keyword I provided and define if this keyword has a name of the known IT software, service, tool or app as a part of it (for example, ServiceNow or Salesforce) and return yes or no."  
      - Prompt type: Define  
      - Output parser enabled for structured JSON response.  
    - Inputs: Receives batches from "Prevent API Rate Limiting" node.  
    - Outputs: Sends parsed AI output to "Update Sheet with Analysis Results".  
    - Version: 1.7  
    - Edge Cases:  
      - AI response may be ambiguous or malformed.  
      - API quota exceeded or authentication failure.  
      - Network timeouts or slow responses.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini model for AI Agent.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Credentials: OpenAI API key (Polina's account)  
    - Inputs: Receives prompt from AI Agent.  
    - Outputs: Returns AI-generated text to AI Agent.  
    - Version: 1.2  
    - Edge Cases:  
      - API key invalid or expired.  
      - Rate limiting or quota exceeded.  
      - Model-specific limitations or errors.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI text response into JSON with schema example: `{ "Isservice": "yes" }`  
    - Configuration: JSON schema example provided to enforce structured output.  
    - Inputs: AI text from OpenAI Chat Model.  
    - Outputs: Parsed JSON to AI Agent.  
    - Version: 1.2  
    - Edge Cases:  
      - Parsing failure if AI output does not match schema.  
      - Unexpected or malformed AI responses.

#### 2.6 Data Update

- **Overview:** Updates the original Google Sheet with the AI classification results in the "Service?" column, matching rows by "Number".
- **Nodes Involved:**  
  - Update Sheet with Analysis Results
- **Node Details:**

  - **Update Sheet with Analysis Results**  
    - Type: Google Sheets  
    - Role: Updates existing rows in the sheet with classification results.  
    - Configuration:  
      - Document ID and Sheet Name same as fetch node.  
      - Operation: Update  
      - Matching column: "Number" (used to identify the correct row)  
      - Columns updated:  
        - "Number": from batch item  
        - "Service?": from AI Agent output JSON field `Isservice`  
      - Other columns present but removed from update.  
      - Credentials: Same Google Sheets OAuth2 account.  
    - Inputs: Receives AI classification results from AI Agent.  
    - Outputs: Sends updated batch data back to "Process Keywords in Batches" to continue processing next batch.  
    - Version: 4.5  
    - Edge Cases:  
      - Row matching failure if "Number" values are missing or duplicated.  
      - Google Sheets API errors or permission issues.  
      - Data type mismatch or update conflicts.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                          | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                  |
|--------------------------------|----------------------------------|----------------------------------------|-------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                   | Starts the workflow manually           | None                          | Fetch Keywords from Sheet        |                                                                                                              |
| Fetch Keywords from Sheet       | Google Sheets                   | Retrieves keywords from Google Sheet   | When clicking ‘Test workflow’ | Process Keywords in Batches      |                                                                                                              |
| Process Keywords in Batches     | SplitInBatches                  | Splits keywords into batches of 6      | Fetch Keywords from Sheet      | Prevent API Rate Limiting (2nd output), (1st output no connection) |                                                                                                              |
| Prevent API Rate Limiting       | Wait                           | Adds delay to prevent API rate limiting| Process Keywords in Batches    | AI Agent                        |                                                                                                              |
| AI Agent                      | Langchain Agent                 | Classifies keywords using AI            | Prevent API Rate Limiting      | Update Sheet with Analysis Results |                                                                                                              |
| OpenAI Chat Model              | Langchain OpenAI Chat Model    | Provides GPT-4o-mini AI model           | AI Agent (ai_languageModel)   | AI Agent (ai_languageModel)      |                                                                                                              |
| Structured Output Parser       | Langchain Output Parser        | Parses AI response into structured JSON| OpenAI Chat Model (ai_outputParser) | AI Agent (ai_outputParser)       |                                                                                                              |
| Update Sheet with Analysis Results | Google Sheets                 | Updates Google Sheet with classification| AI Agent                      | Process Keywords in Batches      |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Google Sheets Node to Fetch Keywords**  
   - Type: Google Sheets  
   - Operation: Read  
   - Document ID: `1jzDvszQoVDV-jrAunCXqTVsiDxXVLMGqQ1zGXwfy5eU`  
   - Sheet Name: `Copy of Sheet1 1` (or sheet ID 1319606837)  
   - Credentials: Connect your Google Sheets OAuth2 account  
   - Connect Manual Trigger output to this node.

3. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Batch Size: 6  
   - Connect output of Google Sheets node to this node.

4. **Create Wait Node for Rate Limiting**  
   - Type: Wait  
   - Default wait time (adjust if needed)  
   - Connect second output of SplitInBatches node (batch continuation) to this node.

5. **Create Langchain OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Credentials: Connect your OpenAI API key  
   - No direct connection yet; will be linked to AI Agent.

6. **Create Langchain Structured Output Parser Node**  
   - Type: Langchain Output Parser Structured  
   - JSON Schema Example:  
     ```json
     {
       "Isservice": "yes"
     }
     ```
   - No direct connection yet; will be linked to AI Agent.

7. **Create Langchain Agent Node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text: `=keyword: {{ $json.Keyword }}`  
     - System Message:  
       `"Check the keyword I provided and define if this keyword has a name of the known IT software, service, tool or app as a part of it (for example, ServiceNow or Salesforce) and return yes or no."`  
     - Prompt Type: Define  
     - Enable Output Parser  
   - Connect:  
     - Input from Wait node (rate limiting)  
     - AI Language Model: Connect to OpenAI Chat Model node  
     - AI Output Parser: Connect to Structured Output Parser node

8. **Create Google Sheets Node to Update Sheet**  
   - Type: Google Sheets  
   - Operation: Update  
   - Document ID and Sheet Name: Same as fetch node  
   - Matching Columns: "Number"  
   - Columns to update:  
     - "Number": `={{ $('Process Keywords in Batches').item.json.Number }}`  
     - "Service?": `={{ $json.output.Isservice }}`  
   - Credentials: Same Google Sheets OAuth2 account  
   - Connect output of AI Agent node to this node.

9. **Connect Update Sheet Node output back to SplitInBatches Node**  
   - This allows the workflow to continue processing the next batch until all keywords are processed.

10. **Verify all connections:**  
    - Manual Trigger → Fetch Keywords from Sheet → Process Keywords in Batches  
    - Process Keywords in Batches (batch continuation) → Prevent API Rate Limiting → AI Agent  
    - AI Agent → Update Sheet with Analysis Results → Process Keywords in Batches (to continue)  
    - AI Agent internal connections: AI Language Model and Output Parser nodes.

11. **Test the workflow** by clicking the manual trigger and monitor for errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The AI prompt is customizable to identify other keyword categories such as educational content, local services, or competitor mentions by modifying the system message in the AI Agent node.                                  | See prompt examples in the workflow description section.                                        |
| Ensure the Google Sheet has columns: "Number", "Keyword", and "Service?" for proper operation.                                                                                                                               |                                                                                                 |
| Adjust batch size in "Process Keywords in Batches" node to balance between API rate limits and processing speed.                                                                                                             |                                                                                                 |
| OpenAI API credentials must have sufficient quota and permissions to use the GPT-4o-mini model.                                                                                                                               |                                                                                                 |
| Google Sheets OAuth2 credentials must have read/write access to the target spreadsheet.                                                                                                                                      |                                                                                                 |
| Screenshot reference available in the original template for visual guidance on sheet structure.                                                                                                                              | File ID: 958                                                                                    |
| For more advanced use cases, consider implementing error handling nodes or retry mechanisms to handle API failures or parsing errors gracefully.                                                                             |                                                                                                 |
| This workflow is a template from n8n community templates repository, designed for SEO and marketing automation.                                                                                                              |                                                                                                 |