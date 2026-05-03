Qualify Leads with AI Review Analysis using Azure GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/qualify-leads-with-ai-review-analysis-using-azure-gpt-4o-mini-and-google-sheets-8278


# Qualify Leads with AI Review Analysis using Azure GPT-4o-mini and Google Sheets

### 1. Workflow Overview

This workflow automates the qualification of leads by analyzing customer product reviews submitted through a Google Sheets form. It integrates AI-driven natural language processing (NLP) using Azure GPT-4o-mini via n8n’s LangChain AI Agent to parse and interpret customer feedback, extracting intent, sentiment, a numerical score, and a summary. The enriched data is then appended or updated in a separate Google Sheets document to maintain a comprehensive and actionable lead database.

**Target Use Cases:**  
- Automating lead qualification from customer reviews or feedback forms  
- Extracting structured insights (intent, sentiment, score, summary) from unstructured text  
- Maintaining an updated, AI-enhanced leads database for sales or marketing teams

**Logical Blocks:**

- **1.1 Input Reception:** Trigger node monitors Google Sheets for new customer reviews.
- **1.2 AI Processing:** AI Agent node processes review text using Azure GPT-4o-mini to analyze content.
- **1.3 Data Parsing:** Code node parses AI JSON output and merges it with original data.
- **1.4 Data Storage:** Google Sheets node appends or updates enriched lead data in a target spreadsheet.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block listens for new entries in a Google Sheets document containing customer reviews. It triggers the workflow automatically every minute when new data appears.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**  
  - **Google Sheets Trigger**  
    - *Type & Role:* Trigger node monitoring spreadsheet changes  
    - *Configuration:*  
      - Poll interval: every minute  
      - Monitors "Form Responses 1" sheet in the "Product Review form (Responses)" Google Sheets document  
      - Uses OAuth2 credentials for access  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Emits new rows as workflow input  
    - *Edge Cases:*  
      - OAuth token expiration or permission errors  
      - API rate limits or network timeouts  
      - Changes in sheet structure (e.g., renamed columns) could cause data extraction issues  
    - *Sticky Note:*  
      Explains node’s role as data source monitor that triggers workflow on new responses.

---

#### Block 1.2: AI Processing

- **Overview:**  
  Processes the customer review text with an AI language model to extract qualitative and quantitative insights for lead qualification.

- **Nodes Involved:**  
  - AI Agent  
  - Azure OpenAI Chat Model

- **Node Details:**  
  - **AI Agent**  
    - *Type & Role:* LangChain AI Agent node for advanced NLP processing  
    - *Configuration:*  
      - Input text: Customer review text from Google Sheets Trigger node  
      - System message instructs the AI to identify intent, sentiment, assign a score (1-10), and summarize the review in JSON format only  
      - Uses "define" prompt type for fixed structured output  
    - *Inputs:* Receives new review text from Google Sheets Trigger node  
    - *Outputs:* JSON string with fields: intent, sentiment, score, summary  
    - *Edge Cases:*  
      - AI service unavailability or timeout  
      - Unexpected AI output format breaking JSON parsing downstream  
      - Limitations on input text length or content quality affecting analysis  
  - **Azure OpenAI Chat Model**  
    - *Type & Role:* AI model connector node providing GPT-4o-mini model  
    - *Configuration:*  
      - Model: gpt-4o-mini  
      - Credentials: Azure OpenAI API key  
    - *Inputs:* Connected as language model provider to AI Agent  
    - *Outputs:* Provides AI-generated content back to AI Agent  
    - *Edge Cases:*  
      - API authentication errors  
      - Model throttling or quota exceeded errors  
      - Network latency causing delays  
    - *Sticky Note:*  
      Details node as the AI language model provider enabling the AI Agent functionality.

---

#### Block 1.3: Data Parsing

- **Overview:**  
  Parses the AI Agent’s JSON output and merges extracted insights with the original customer review data for downstream usage.

- **Nodes Involved:**  
  - Code - Parse AI JSON

- **Node Details:**  
  - **Code - Parse AI JSON**  
    - *Type & Role:* JavaScript code node for data transformation and parsing  
    - *Configuration:*  
      - Iterates over all input items  
      - Parses the AI Agent’s stringified JSON output  
      - Extracts keys: intent, sentiment, score, summary  
      - Merges AI data with original sheet fields: Timestamp, Name, Email, Contact Number, Review of the product  
      - Removes temporary fields used during processing  
    - *Inputs:* Receives AI Agent output enriched with original sheet data  
    - *Outputs:* Items with combined, clean data ready for storage  
    - *Edge Cases:*  
      - Malformed or invalid JSON in AI output causing parse errors  
      - Missing expected fields in original sheet data  
      - Runtime errors in custom JavaScript code  
    - *Sticky Note:*  
      Explains this node’s role as data parser and formatter, preparing data for storage.

---

#### Block 1.4: Data Storage

- **Overview:**  
  Stores the enriched lead information into a specified Google Sheets document, either updating existing rows or appending new entries.

- **Nodes Involved:**  
  - Google Sheets - Update Lead

- **Node Details:**  
  - **Google Sheets - Update Lead**  
    - *Type & Role:* Google Sheets node for data writing  
    - *Configuration:*  
      - Document: "Updated product review" spreadsheet  
      - Sheet: "Sheet1" (gid=0)  
      - Operation: Append or update rows based on a matching `row_number` field  
      - Columns mapped include all original data plus AI fields (Intent, Sentiment, Score, Summary)  
      - Uses OAuth2 credentials for write access  
    - *Inputs:* Processed data from Code node  
    - *Outputs:* None (terminal node in this workflow)  
    - *Edge Cases:*  
      - Authentication or permission errors  
      - Conflicts or failures in row matching logic  
      - API rate limits or network errors during write operations  
    - *Sticky Note:*  
      Clarifies node’s purpose to save combined customer and AI data, maintaining an enhanced feedback database.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role          | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                    |
|-------------------------|-------------------------------------|-------------------------|----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger    | n8n-nodes-base.googleSheetsTrigger  | Input Reception         | -                          | AI Agent                   | This node continuously monitors a spreadsheet for new customer review submissions.                             |
| AI Agent                | @n8n/n8n-nodes-langchain.agent       | AI Processing           | Google Sheets Trigger       | Code - Parse AI JSON       | Processes reviews with AI to extract intent, sentiment, score, and summary in JSON format.                    |
| Azure OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | AI Model Provider       | -                          | AI Agent (as language model) | Provides GPT-4o-mini model access to AI Agent for natural language processing.                                |
| Code - Parse AI JSON     | n8n-nodes-base.code                  | Data Parsing            | AI Agent                   | Google Sheets - Update Lead | Parses AI JSON output, merges with original data, and prepares for storage.                                   |
| Google Sheets - Update Lead | n8n-nodes-base.googleSheets         | Data Storage            | Code - Parse AI JSON       | -                          | Saves enriched lead data back into Google Sheets, appending or updating rows accordingly.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Type: Google Sheets Trigger  
   - Configure to poll the spreadsheet "Product Review form (Responses)" every minute  
   - Select sheet named "Form Responses 1" (gid=931869588)  
   - Set OAuth2 credentials for Google Sheets Trigger API access  
   - Purpose: Automatically trigger workflow on new form responses

2. **Create Azure OpenAI Chat Model Node:**  
   - Type: LangChain Azure OpenAI Chat Model  
   - Select model: "gpt-4o-mini"  
   - Set Azure OpenAI credentials with valid API key  
   - Purpose: Provide GPT-4o-mini model for AI processing

3. **Create AI Agent Node:**  
   - Type: LangChain AI Agent  
   - Input Text expression:  
     `=Here is the customer review data: {{ $json["Review of the product"] }}`  
   - System Message (as prompt):  
     ```
     You are an AI assistant that analyzes customer product reviews.
     Your job is to:
     
     Understand the customer’s intent (e.g., praise, complaint, suggestion, feature request, mixed feedback).
     
     Evaluate the sentiment (positive, neutral, negative, or mixed).
     
     Assign a score from 1–10 (1 = very negative, 10 = excellent).
     
     Return results in clean JSON format with these keys:
     
     intent
     
     sentiment
     
     score
     
     summary (short 1–2 sentence summary of the review).
     
     Do not add extra commentary outside the JSON.
     ```
   - Prompt type: Define (fixed structured output)  
   - Link the AI Agent’s language model to the Azure OpenAI Chat Model node  
   - Purpose: Analyze review text and generate structured AI insights

4. **Create Code Node - Parse AI JSON:**  
   - Type: Code (JavaScript)  
   - Paste the following code:  
     ```javascript
     for (const item of $input.all()) {
       const parsed = JSON.parse(item.json.output);
       const sheet = $('Google Sheets Trigger').first().json;
       
       item.json.Timestamp = sheet.Timestamp;
       item.json.Name = sheet.Name;
       item.json.Email = sheet.Email;
       item.json['Contact Number'] = sheet['Contact Number'];
       item.json['Review of the product'] = sheet['Review of the product'];
       item.json.intent = parsed.intent;
       item.json.sentiment = parsed.sentiment;
       item.json.score = parsed.score;
       item.json.summary = parsed.summary;
       
       delete item.json.output;
     }
     return $input.all();
     ```  
   - Purpose: Parse AI JSON output and merge with original sheet data

5. **Create Google Sheets Node - Update Lead:**  
   - Type: Google Sheets  
   - Document: "Updated product review" spreadsheet (ID: `1qohS8H6x3_CrXiT0t1jpxjDk_YbEbHrkAa2tDHWJCc4`)  
   - Sheet: "Sheet1" (gid=0)  
   - Operation: Append or Update  
   - Matching column: `row_number` (to update existing rows if applicable)  
   - Map columns: Timestamp, Name, Email, Contact Number, Review of the product, Intent, Sentiment, Score, Summary  
   - Set OAuth2 credentials for Google Sheets API access  
   - Purpose: Persist AI-enriched lead data back into a maintainable spreadsheet

6. **Connect the Nodes in Order:**  
   - Google Sheets Trigger → AI Agent  
   - AI Agent → Code - Parse AI JSON  
   - Code - Parse AI JSON → Google Sheets - Update Lead  
   - Azure OpenAI Chat Model → AI Agent (as language model provider)

7. **Credentials Setup:**  
   - Google Sheets Trigger: OAuth2 account with read access to source sheet  
   - Google Sheets Update Lead: OAuth2 account with write access to destination sheet  
   - Azure OpenAI Chat Model: Azure OpenAI API key with GPT-4o-mini access

8. **Test the Workflow:**  
   - Submit a new entry in the source Google Sheet form  
   - Verify that workflow triggers, AI processes data, and results appear in the target sheet

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| The AI Agent node uses LangChain integration in n8n, which requires n8n version supporting AI nodes. | Refer to n8n documentation on AI integrations for setup details.                                               |
| GPT-4o-mini model is an Azure OpenAI offering optimized for cost-effective natural language tasks. | Azure OpenAI model details: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/concepts/models |
| Google Sheets API OAuth2 credentials must have proper scopes to read/write respective spreadsheets. | Google Cloud Console: https://console.cloud.google.com/apis/credentials                                         |
| To avoid JSON parsing errors, ensure AI system prompt enforces strict JSON-only output.           | Use "define" prompt type in AI Agent for consistent structured output                                          |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.