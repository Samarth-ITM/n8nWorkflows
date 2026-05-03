Generate Sales Reports from Google Sheets with Azure OpenAI and Telegram

https://n8nworkflows.xyz/workflows/generate-sales-reports-from-google-sheets-with-azure-openai-and-telegram-8177


# Generate Sales Reports from Google Sheets with Azure OpenAI and Telegram

---

### 1. Workflow Overview

This workflow automates the generation and delivery of sales reports by extracting data from Google Sheets, summarizing it using Azure OpenAI’s language model, and sending the resulting summary via Telegram. It is designed for sales teams or managers who want to receive concise, AI-generated insights from sales data stored in spreadsheets, directly on their messaging app.

The workflow is structured in the following logical blocks:

- **1.1 Input Trigger and Data Retrieval:** Initiates the workflow manually and fetches sales data rows from a Google Sheet.
- **1.2 AI Processing:** Processes the retrieved data through an Azure OpenAI chat model using a LangChain LLM node to generate a summary.
- **1.3 Output Delivery:** Sends the AI-generated sales report summary as a Telegram text message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Data Retrieval

**Overview:**  
This block starts the workflow manually and retrieves necessary sales data from a Google Sheets spreadsheet to be processed.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet (Google Sheets)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - *Type & Role:* Manual Trigger node; initiates the workflow on user command.  
  - *Configuration:* No parameters needed; waits for manual execution.  
  - *Inputs/Outputs:* No input; outputs trigger signal to ‘Get row(s) in sheet’.  
  - *Failures/Edge Cases:* None; manual start avoids automatic errors but requires user interaction.

- **Get row(s) in sheet**  
  - *Type & Role:* Google Sheets node; reads data rows from a specified Google Sheet document.  
  - *Configuration:* Parameters not explicitly provided in JSON, but typically includes spreadsheet ID, sheet name, and row range.  
  - *Inputs/Outputs:* Input from manual trigger; outputs rows of sales data to ‘Basic LLM Chain’.  
  - *Failures/Edge Cases:* Possible authentication errors if Google API credentials are invalid; empty or malformed sheets may produce no data or errors; API rate limits may apply.

---

#### 2.2 AI Processing

**Overview:**  
This block takes the raw sales data and uses Azure OpenAI’s chat model wrapped by LangChain to generate a natural language summary report.

**Nodes Involved:**  
- Azure OpenAI Chat Model  
- Basic LLM Chain

**Node Details:**  

- **Azure OpenAI Chat Model**  
  - *Type & Role:* LangChain Azure OpenAI Chat Model node; provides the underlying AI language model service.  
  - *Configuration:* Utilizes Azure OpenAI credentials; model parameters (specific model, temperature, token limits) are configured inside this node or inherited by the chain node.  
  - *Inputs/Outputs:* Feeds AI language model to ‘Basic LLM Chain’ via ai_languageModel connection.  
  - *Failures/Edge Cases:* Authentication failures with Azure credentials; API timeouts; exceeding token or rate limits; model unavailability.

- **Basic LLM Chain**  
  - *Type & Role:* LangChain Chain node; orchestrates input data into prompts and sends them to Azure OpenAI Chat Model to get a summarized text.  
  - *Configuration:* Contains prompt template or chain logic to convert spreadsheet data into a sales report summary.  
  - *Inputs/Outputs:* Receives raw spreadsheet rows from ‘Get row(s) in sheet’; connects to Azure OpenAI Chat Model as language model; outputs summary text to ‘Send a text message’.  
  - *Failures/Edge Cases:* Expression or prompt formatting errors; unexpected input data shape; handling empty data gracefully.

---

#### 2.3 Output Delivery

**Overview:**  
This block sends the AI-generated sales summary to a Telegram user or channel.

**Nodes Involved:**  
- Send a text message (Telegram)

**Node Details:**  

- **Send a text message**  
  - *Type & Role:* Telegram node; sends text messages via Telegram Bot API.  
  - *Configuration:* Requires Telegram Bot credentials; message content is set dynamically from the output of ‘Basic LLM Chain’.  
  - *Inputs/Outputs:* Receives summary text from ‘Basic LLM Chain’; outputs confirmation of message sent (not connected further).  
  - *Failures/Edge Cases:* Invalid Telegram credentials; message formatting errors; network/API downtime; blocked bot or chat ID issues.

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                     | Input Node(s)             | Output Node(s)          | Sticky Note                          |
|---------------------------|---------------------------------------|-----------------------------------|---------------------------|-------------------------|------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                         | Workflow start trigger             | —                         | Get row(s) in sheet      |                                    |
| Get row(s) in sheet       | Google Sheets                         | Retrieve sales data from spreadsheet | When clicking ‘Execute workflow’ | Basic LLM Chain          |                                    |
| Azure OpenAI Chat Model   | LangChain Azure OpenAI Chat Model    | AI language model backend          | — (used by Basic LLM Chain) | Basic LLM Chain (ai_languageModel) |                                    |
| Basic LLM Chain           | LangChain Chain                      | Process data with AI to generate summary | Get row(s) in sheet, Azure OpenAI Chat Model | Send a text message       |                                    |
| Send a text message       | Telegram                             | Deliver summarized sales report    | Basic LLM Chain            | —                       |                                    |
| Sticky Note               | Sticky Note                         | —                                 | —                         | —                       |                                    |
| Sticky Note1              | Sticky Note                         | —                                 | —                         | —                       |                                    |
| Sticky Note2              | Sticky Note                         | —                                 | —                         | —                       |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Add a Google Sheets node**  
   - Name: `Get row(s) in sheet`  
   - Credentials: Configure with Google Sheets OAuth2 credentials.  
   - Parameters: Specify the spreadsheet ID, sheet name, and range of rows to fetch sales data (e.g., all rows or a specific range).  
   - Connect output of `When clicking ‘Execute workflow’` to this node.

3. **Add Azure OpenAI Chat Model node (LangChain)**  
   - Name: `Azure OpenAI Chat Model`  
   - Credentials: Configure with Azure OpenAI API key and endpoint.  
   - Configure model details (e.g., GPT-4 or GPT-3.5) and other parameters like temperature, max tokens.  
   - This node provides the language model to the chain node.

4. **Add Basic LLM Chain node (LangChain)**  
   - Name: `Basic LLM Chain`  
   - Inputs:  
     - Connect input from `Get row(s) in sheet` to receive sales data.  
     - Connect `ai_languageModel` input to `Azure OpenAI Chat Model` node.  
   - Configuration:  
     - Define prompt template or chain logic to convert spreadsheet rows into a natural language sales summary.  
     - Use expressions to map spreadsheet data dynamically into the prompt.  
   - Connect output to the Telegram node.

5. **Add Telegram node**  
   - Name: `Send a text message`  
   - Credentials: Configure with Telegram Bot credentials (bot token).  
   - Parameters:  
     - Set chat ID or recipient.  
     - Set message text dynamically using the output from `Basic LLM Chain`.  
   - Connect input from `Basic LLM Chain`.

6. **Save and activate the workflow**  
   - Test by manually executing the trigger node.  
   - Verify data retrieval, AI summary generation, and Telegram message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                          |
|---------------------------------------------------------------------------------------------------------|----------------------------------------|
| n8n LangChain nodes require LangChain package installed and credentials for Azure OpenAI configured.    | https://docs.n8n.io/integrations/builtin/ai/langchain/ |
| Telegram node requires proper bot creation and chat ID configuration.                                   | https://core.telegram.org/bots/api     |
| Google Sheets node requires OAuth2 credentials with permission to read the target spreadsheet.          | https://developers.google.com/sheets/api/guides/authorizing#oauth2 |
| This workflow uses manual trigger to avoid unintended automatic runs; suitable for on-demand reporting. |                                        |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---