Stock Fundamental Analysis & AI-Powered Reports with Mistral and AlphaVantage

https://n8nworkflows.xyz/workflows/stock-fundamental-analysis---ai-powered-reports-with-mistral-and-alphavantage-5710


# Stock Fundamental Analysis & AI-Powered Reports with Mistral and AlphaVantage

---

## 1. Workflow Overview

**Purpose:**  
This workflow automates stock fundamental analysis and generates AI-powered reports by integrating financial data retrieval, news aggregation, and advanced language model processing. It targets users who want to automatically compile comprehensive stock reports combining data from AlphaVantage and news sources, augmented by AI summarization and structuring via Mistral language models.

**Target Use Cases:**  
- Automated generation of detailed stock fundamental analysis reports.  
- Integration of real-time news data with financial metrics.  
- Delivery of AI-enhanced summaries and insights via email.  
- Handling form submissions to trigger the workflow with dynamic stock inputs.

**Logical Blocks:**

- **1.1 Input Reception**  
  Handles form submissions to collect user inputs (e.g., stock symbols, parameters).

- **1.2 Data Retrieval**  
  Fetches stock-related news from multiple HTTP API requests, consolidates outputs.

- **1.3 Data Processing & Transformation**  
  Splits, limits, aggregates, and merges the news data for further processing.

- **1.4 AI Model Processing**  
  Uses multiple Mistral cloud chat models and LangChain output parsers to generate structured, auto-fixed AI summaries and reports based on the gathered data.

- **1.5 Report Formatting and Distribution**  
  Converts AI outputs into HTML format and sends the final report via Gmail.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block captures stock analysis requests through a form submission trigger, initializing variables for subsequent data retrieval.

**Nodes Involved:**  
- On form submission  
- Set Variables

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point receiving user inputs dynamically.  
  - *Config:* Uses webhook ID `cbf3d8a9-db36-414a-b9fb-3d764b59d923`.  
  - *Input:* HTTP webhook request triggered by user form submission.  
  - *Output:* Passes form data to the next node.  
  - *Failure Modes:* Webhook delivery issues, malformed submissions.

- **Set Variables**  
  - *Type:* Set  
  - *Role:* Initializes or formats variables based on form data.  
  - *Config:* No explicit parameters set in JSON; likely sets variables dynamically from form payload.  
  - *Input:* Data from form submission trigger.  
  - *Output:* Feeds multiple HTTP request nodes for news data retrieval.  
  - *Failure Modes:* Missing or invalid form fields leading to empty or incorrect variables.

---

### 2.2 Data Retrieval

**Overview:**  
This block performs multiple HTTP requests in parallel to fetch news data relevant to the stock(s) being analyzed, then merges the results into a single data stream.

**Nodes Involved:**  
- Get News Data  
- Get News Data1  
- Get News Data2  
- Get News Data3  
- Get News Data4  
- Get News Data5  
- Merge

**Node Details:**

- **Get News Data / Get News Data1 / Get News Data2 / Get News Data3 / Get News Data4 / Get News Data5**  
  - *Type:* HTTP Request  
  - *Role:* Fetch news data from external API endpoints, presumably different queries or sources.  
  - *Config:* Parameters not explicitly detailed but likely include API URLs, authentication, and query parameters for stock news.  
  - *Input:* Variables from "Set Variables" node.  
  - *Output:* News data JSON or similar format.  
  - *Failure Modes:* HTTP errors, rate limits, malformed responses, API key issues.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines multiple news data streams into one unified dataset.  
  - *Config:* Defaults; merges all incoming inputs.  
  - *Input:* Six HTTP request nodes outputs.  
  - *Output:* Consolidated news data array.  
  - *Failure Modes:* Data format inconsistencies; merging empty inputs.

---

### 2.3 Data Processing & Transformation

**Overview:**  
This block splits the consolidated news data for granular processing, limits the dataset size, aggregates the data, and prepares for AI model input.

**Nodes Involved:**  
- Split Out  
- Split Out2  
- Merge1  
- Limit  
- Aggregate  
- Merge2  
- Code2  
- Code1

**Node Details:**

- **Split Out & Split Out2**  
  - *Type:* Split Out  
  - *Role:* Divides merged news data into smaller chunks or separate items for individual processing paths.  
  - *Input:* Output of Merge node.  
  - *Output:* Two separate streams feeding into Merge1.  
  - *Failure Modes:* Empty inputs or unexpected data structure causing split failures.

- **Merge1**  
  - *Type:* Merge  
  - *Role:* Combines the two split streams back into one, possibly interleaving or consolidating.  
  - *Input:* Split Out and Split Out2 outputs.  
  - *Output:* To Limit node.  
  - *Failure Modes:* Data ordering or duplication issues.

- **Limit**  
  - *Type:* Limit  
  - *Role:* Restricts number of items passing downstream to avoid overload or API limits.  
  - *Config:* Default limit parameters (not specified).  
  - *Input:* Output from Merge1.  
  - *Output:* To Aggregate.  
  - *Failure Modes:* Limit too low or too high impacting downstream processing.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Aggregates limited news data into a single collection object for AI input.  
  - *Input:* Output from Limit.  
  - *Output:* To Merge2 node.  
  - *Failure Modes:* Aggregation errors if input data malformed.

- **Merge2**  
  - *Type:* Merge  
  - *Role:* Combines aggregated news data with additional processed data from Code2.  
  - *Input:* Aggregate output and Code2 output.  
  - *Output:* To Code1.  
  - *Failure Modes:* Data inconsistency or missing input issues.

- **Code2**  
  - *Type:* Code (JavaScript)  
  - *Role:* Custom data transformation or formatting on news data or variables before merging.  
  - *Input:* One of the split news data streams.  
  - *Output:* To Merge2.  
  - *Failure Modes:* Runtime JavaScript errors, undefined variables.

- **Code1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Final data preparation or restructuring before AI chain processing.  
  - *Input:* Output from Merge2.  
  - *Output:* To Mistral Cloud Chat Model1.  
  - *Failure Modes:* Script errors or data format issues.

---

### 2.4 AI Model Processing

**Overview:**  
This block applies advanced AI models to the prepared data for generating structured and auto-corrected natural language summaries and reports.

**Nodes Involved:**  
- Mistral Cloud Chat Model1  
- Basic LLM Chain  
- HTML  
- Mistral Cloud Chat Model  
- Auto-fixing Output Parser  
- Structured Output Parser1  
- Basic LLM Chain1  
- HTML2  
- Mistral Cloud Chat Model2  
- Structured Output Parser

**Node Details:**

- **Mistral Cloud Chat Model1**  
  - *Type:* AI Language Model (Mistral)  
  - *Role:* Primary AI model to process prepared stock data for initial generation.  
  - *Input:* Output from Code1.  
  - *Output:* To Basic LLM Chain.  
  - *Failure Modes:* API auth errors, rate limits, response timeouts.

- **Basic LLM Chain**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Processes AI model output through LangChain for chaining logic and prompt management.  
  - *Input:* Output from Mistral Cloud Chat Model1.  
  - *Output:* To HTML node.  
  - *Failure Modes:* Chain configuration errors, prompt failures.

- **HTML**  
  - *Type:* HTML  
  - *Role:* Converts AI output text into formatted HTML for report generation.  
  - *Input:* Output from Basic LLM Chain.  
  - *Output:* To Basic LLM Chain1.  
  - *Failure Modes:* HTML formatting errors.

- **Basic LLM Chain1**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Further AI processing, likely for report enhancement or refinement.  
  - *Input:* Output from HTML.  
  - *Output:* To HTML2.  
  - *Failure Modes:* Same as Basic LLM Chain.

- **HTML2**  
  - *Type:* HTML  
  - *Role:* Final HTML formatting of the AI-generated report.  
  - *Input:* Output from Basic LLM Chain1.  
  - *Output:* To Gmail node for emailing.  
  - *Failure Modes:* Formatting errors.

- **Mistral Cloud Chat Model**  
  - *Type:* AI Language Model  
  - *Role:* AI model used in parallel or cascade with other models for auto-fixing output.  
  - *Input:* Output from Structured Output Parser1.  
  - *Output:* To Auto-fixing Output Parser.  
  - *Failure Modes:* Same as other Mistral nodes.

- **Auto-fixing Output Parser**  
  - *Type:* LangChain Output Parser (Auto-fixing)  
  - *Role:* Automatically corrects and refines AI output parsing errors.  
  - *Input:* Output from Mistral Cloud Chat Model.  
  - *Output:* To Basic LLM Chain.  
  - *Failure Modes:* Parsing failures, unhandled output formats.

- **Structured Output Parser1**  
  - *Type:* LangChain Output Parser (Structured)  
  - *Role:* Parses AI output into structured data for further AI processing.  
  - *Input:* Output from Basic LLM Chain1.  
  - *Output:* To Mistral Cloud Chat Model.  
  - *Failure Modes:* Parsing errors.

- **Mistral Cloud Chat Model2**  
  - *Type:* AI Language Model  
  - *Role:* Additional AI processing in the chain, possibly for final summarization.  
  - *Input:* Output from Basic LLM Chain1.  
  - *Output:* To Structured Output Parser.  
  - *Failure Modes:* Same as other Mistral nodes.

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser (Structured)  
  - *Role:* Final structured parsing before report formatting.  
  - *Input:* Output from Mistral Cloud Chat Model2.  
  - *Output:* To Basic LLM Chain1.  
  - *Failure Modes:* Parsing failures.

---

### 2.5 Report Formatting and Distribution

**Overview:**  
Finalizes the generated stock report as HTML and sends it via Gmail to intended recipients.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - *Type:* Gmail node  
  - *Role:* Sends the formatted email report via the user's Gmail account.  
  - *Config:* Uses OAuth2 credentials, webhook ID `8eae312c-743f-4238-b949-7539099096b0`.  
  - *Input:* HTML-formatted report from HTML2 node.  
  - *Output:* Email sent confirmation or error.  
  - *Failure Modes:* Authentication errors, quota exceeded, network issues.

---

## 3. Summary Table

| Node Name               | Node Type                            | Functional Role                       | Input Node(s)                       | Output Node(s)                    | Sticky Note               |
|-------------------------|------------------------------------|------------------------------------|-----------------------------------|---------------------------------|---------------------------|
| On form submission      | Form Trigger                       | Entry point — receives user input   | -                                 | Set Variables                   |                           |
| Set Variables           | Set                                | Initialize variables from form data | On form submission                | Get News Data, Get News Data1,...|                           |
| Get News Data           | HTTP Request                      | Fetch news data from API             | Set Variables                    | Merge                          |                           |
| Get News Data1          | HTTP Request                      | Fetch news data from API             | Set Variables                    | Merge                          |                           |
| Get News Data2          | HTTP Request                      | Fetch news data from API             | Set Variables                    | Merge                          |                           |
| Get News Data3          | HTTP Request                      | Fetch news data from API             | Set Variables                    | Merge                          |                           |
| Get News Data4          | HTTP Request                      | Fetch news data from API             | Set Variables                    | Merge                          |                           |
| Get News Data5          | HTTP Request                      | Fetch news data from API             | Set Variables                    | Merge                          |                           |
| Merge                   | Merge                             | Combine multiple news data streams  | Get News Data, ..., Get News Data5 | Split Out, Split Out2, Code2    |                           |
| Split Out               | Split Out                        | Split merged news data              | Merge                           | Merge1                         |                           |
| Split Out2              | Split Out                        | Split merged news data              | Merge                           | Merge1                         |                           |
| Merge1                  | Merge                             | Combine split streams               | Split Out, Split Out2             | Limit                         |                           |
| Limit                   | Limit                             | Limit number of news data items     | Merge1                          | Aggregate                     |                           |
| Aggregate               | Aggregate                        | Aggregate limited data              | Limit                          | Merge2                        |                           |
| Merge2                  | Merge                             | Combine aggregated data             | Aggregate, Code2                 | Code1                         |                           |
| Code2                   | Code                              | Custom data transformation          | Merge                           | Merge2                        |                           |
| Code1                   | Code                              | Final data preparation              | Merge2                         | Mistral Cloud Chat Model1       |                           |
| Mistral Cloud Chat Model1| AI Language Model (Mistral)        | Generate AI analysis output         | Code1                          | Basic LLM Chain               |                           |
| Basic LLM Chain         | LangChain LLM Chain               | Manage AI chain processing          | Mistral Cloud Chat Model1        | HTML                         |                           |
| HTML                    | HTML                              | Format text as HTML                  | Basic LLM Chain                 | Basic LLM Chain1              |                           |
| Basic LLM Chain1        | LangChain LLM Chain               | Further AI processing                | HTML                          | HTML2                        |                           |
| HTML2                   | HTML                              | Final HTML formatting                | Basic LLM Chain1               | Gmail                        |                           |
| Gmail                   | Gmail                             | Send email report                   | HTML2                         | -                            |                           |
| Mistral Cloud Chat Model | AI Language Model (Mistral)        | Auto-fixing AI output                | Structured Output Parser1       | Auto-fixing Output Parser    |                           |
| Auto-fixing Output Parser| LangChain Output Parser (Auto-fixing)| Auto-correct AI output parsing     | Mistral Cloud Chat Model        | Basic LLM Chain              |                           |
| Structured Output Parser1| LangChain Output Parser (Structured)| Structured parsing of AI output     | Basic LLM Chain1               | Mistral Cloud Chat Model     |                           |
| Mistral Cloud Chat Model2| AI Language Model (Mistral)        | Additional AI output generation      | Basic LLM Chain1               | Structured Output Parser     |                           |
| Structured Output Parser | LangChain Output Parser (Structured)| Final structured parsing            | Mistral Cloud Chat Model2       | Basic LLM Chain1              |                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: `Form Trigger`  
   - Configure webhook ID (auto-generated or manually set).  
   - Purpose: Entry point to receive stock symbol or analysis parameters.

2. **Add a Set node:**  
   - Type: `Set`  
   - Configure to initialize variables based on form submission data (e.g., stock symbol, date ranges).  
   - Connect: From Form Trigger node.

3. **Add six HTTP Request nodes for news data:**  
   - Names: Get News Data, Get News Data1, ..., Get News Data5  
   - Configure each with appropriate API endpoint URLs, authentication (e.g., API keys), and query parameters for news retrieval related to the stock symbol.  
   - Connect: Each from Set Variables node.

4. **Add a Merge node:**  
   - Type: `Merge`  
   - Configure to merge all six HTTP requests outputs into one stream.  
   - Connect: Inputs from all six HTTP Request nodes.

5. **Add two Split Out nodes:**  
   - Type: `Split Out`  
   - Connect: Both from the Merge node output.

6. **Add a Merge node (Merge1):**  
   - Type: `Merge`  
   - Connect: Inputs from the two Split Out nodes.

7. **Add a Limit node:**  
   - Type: `Limit`  
   - Set limit for maximum items to process (default or custom).  
   - Connect: From Merge1.

8. **Add an Aggregate node:**  
   - Type: `Aggregate`  
   - Configure to collect limited items into a single collection.  
   - Connect: From Limit node.

9. **Add a Merge node (Merge2):**  
   - Type: `Merge`  
   - Connect inputs from Aggregate node and Code2 node (to be created next).

10. **Add Code nodes:**  
    - Code2: Custom JavaScript to transform or format data from Merge node before merging. Connect output to Merge2.  
    - Code1: Final JavaScript data preparation before AI processing. Connect input from Merge2 and output to Mistral Cloud Chat Model1.

11. **Add Mistral Cloud Chat Model1 node:**  
    - Type: AI Language Model (Mistral)  
    - Configure with Mistral API credentials.  
    - Connect: From Code1 node output.

12. **Add Basic LLM Chain node:**  
    - Type: LangChain LLM Chain  
    - Configure with appropriate prompt templates and chain logic.  
    - Connect: From Mistral Cloud Chat Model1.

13. **Add HTML node:**  
    - Convert AI output into HTML format.  
    - Connect: From Basic LLM Chain.

14. **Add Basic LLM Chain1 node:**  
    - Further AI processing for refinement.  
    - Connect: From HTML node.

15. **Add HTML2 node:**  
    - Final HTML formatting.  
    - Connect: From Basic LLM Chain1.

16. **Add Gmail node:**  
    - Configure Gmail OAuth2 credentials for sending email.  
    - Connect: From HTML2 node.  
    - Set email parameters: recipient, subject, body (HTML).

17. **Add Mistral Cloud Chat Model node:**  
    - Type: AI Language Model  
    - Connect: From Structured Output Parser1.

18. **Add Auto-fixing Output Parser node:**  
    - Type: LangChain Auto-fixing Output Parser  
    - Connect: From Mistral Cloud Chat Model node output to Basic LLM Chain.

19. **Add Structured Output Parser1 node:**  
    - Type: LangChain Structured Output Parser  
    - Connect: From Basic LLM Chain1 output to Mistral Cloud Chat Model.

20. **Add Mistral Cloud Chat Model2 node:**  
    - Connect: From Basic LLM Chain1 output.

21. **Add Structured Output Parser node:**  
    - Final structured output parsing.  
    - Connect: From Mistral Cloud Chat Model2 output to Basic LLM Chain1.

22. **Verify connections and test the entire flow:**  
    - Ensure all API credentials are properly configured (AlphaVantage, Mistral API, Gmail OAuth2).  
    - Test with sample stock inputs via form submission.  
    - Monitor logs for errors and adjust limits or parsing logic as needed.

---

## 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow combines AlphaVantage API data with multiple news sources to enrich stock fundamental analysis.         | Use of multiple HTTP Request nodes fetching diversified news data streams.                       |
| Mistral Cloud Chat Models are used as AI engines to generate and auto-correct stock analysis reports.                 | Requires valid Mistral API credentials.                                                         |
| Gmail node requires OAuth2 setup to send emails securely with proper authorization.                                    | Refer to n8n Gmail node OAuth2 documentation for setup.                                          |
| The workflow is triggered by a form submission webhook, suitable for integration with n8n's built-in form or external. | Webhook ID: cbf3d8a9-db36-414a-b9fb-3d764b59d923                                               |
| Output parsers from LangChain (Structured, Auto-fixing) ensure AI output reliability and format consistency.          | n8n LangChain nodes documentation: https://docs.n8n.io/integrations/builtin/nodes/langchain/    |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and publicly available.

---