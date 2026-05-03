Generate Investment Recommendations from Stock Data using Finnhub and Gemini 2.5-Flash

https://n8nworkflows.xyz/workflows/generate-investment-recommendations-from-stock-data-using-finnhub-and-gemini-2-5-flash-11315


# Generate Investment Recommendations from Stock Data using Finnhub and Gemini 2.5-Flash

### 1. Workflow Overview

This workflow, titled **"Fundamental Stock Analysis Utilizing Gemini 2.5-Flash"**, is designed to generate investment recommendations based on fundamental stock data. It integrates financial data from external APIs (presumably Finnhub or similar sources) and leverages Google Gemini AI models for advanced stock analysis and recommendation generation.

The workflow targets investment analysts, portfolio managers, and automated trading systems that require enriched fundamental data combined with AI-driven insights.

The logic is grouped into two main parallel processing blocks (conditional on the type of most recent company filing) and a final AI analysis and report generation block:

- **1.1 Input Reception**: Receives user input via two form triggers, determining whether the most recent company filing is a quarterly or annual report.
- **1.2 Data Acquisition and Preprocessing**: Fetches relevant financial data (Ratios, Financials, Market Cap, Quotes) using HTTP requests and filters important metrics.
- **1.3 Data Aggregation and Calculations**: Aggregates filtered financial data, calculates trailing twelve months (TTM), Compound Annual Growth Rate (CAGR), and prediction models.
- **1.4 AI Processing and Reporting**: Uses Google Gemini Chat Models and LangChain agents to analyze aggregated data and generate investment recommendations, converting outputs to HTML and file formats, and saving reports locally.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block consists of two form trigger nodes that act as entry points. One triggers when the most recent filing is a quarterly report, the other for an annual report. Each triggers a different data retrieval and processing path.

**Nodes Involved:**  
- Use if most recent company filing is quarterly report  
- Use if most recent filing is annual report  

**Node Details:**  

- **Use if most recent company filing is quarterly report**  
  - *Type:* Form Trigger (Webhook)  
  - *Role:* Starting point for quarterly report processing  
  - *Configuration:* Webhook ID assigned; waits for form submission to start workflow  
  - *Input/Output:* No input; outputs to Quote, Financials, Ratios, and MarketCap HTTP request nodes  
  - *Edge Cases:* Webhook failures, invalid input, or missing required fields may cause stoppage  
  - *Version:* 2.3  

- **Use if most recent filing is annual report**  
  - *Type:* Form Trigger (Webhook)  
  - *Role:* Starting point for annual report processing  
  - *Configuration:* Webhook ID assigned; waits for form submission to start workflow  
  - *Input/Output:* No input; outputs to Quote1, Financials1, Ratios1 HTTP request nodes  
  - *Edge Cases:* Similar to above; webhook errors or invalid data may interrupt flow  
  - *Version:* 2.3  

---

#### 2.2 Data Acquisition and Preprocessing

**Overview:**  
This block fetches financial data from external APIs and filters for relevant key metrics. It separates the data streams depending on the filing type, ensuring correct data handling for quarterly and annual reports.

**Nodes Involved (Quarterly path):**  
- Ratios  
- Financials  
- MarketCap  
- Quote  
- Filter Important Ratios  
- Filter Important Financials  

**Nodes Involved (Annual path):**  
- Ratios1  
- Financials1  
- Quote1  
- Filter Important Ratios1  
- Filter Important Financials1  

**Node Details:**

- **Ratios / Ratios1**  
  - *Type:* HTTP Request  
  - *Role:* Retrieve financial ratios from API  
  - *Config:* Presumably configured with API endpoint and credentials (not shown)  
  - *Output:* Raw ratios data to filtering code nodes  
  - *Edge Cases:* API timeout, invalid API key, rate limits  

- **Financials / Financials1**  
  - *Type:* HTTP Request  
  - *Role:* Retrieve financial statement data (balance sheet, income statement, etc.)  
  - *Config:* As above, configured with API endpoint and credentials  
  - *Output:* Raw financial data to filtering code nodes  
  - *Edge Cases:* Data missing or incomplete, API errors  

- **MarketCap**  
  - *Type:* HTTP Request  
  - *Role:* Retrieve market capitalization data (only used in quarterly path)  
  - *Output:* Market cap data merged with other data streams  
  - *Edge Cases:* API errors or missing data  

- **Quote / Quote1**  
  - *Type:* HTTP Request  
  - *Role:* Retrieve current stock price and quote data  
  - *Output:* Feeds into merge nodes for aggregation  
  - *Edge Cases:* API rate limits, invalid symbol errors  

- **Filter Important Ratios / Filter Important Ratios1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filter raw ratios to keep only key financial ratios relevant for analysis  
  - *Inputs:* Raw ratios data  
  - *Outputs:* Filtered ratios data to merge nodes  
  - *Edge Cases:* Expression failures or missing expected properties in data  

- **Filter Important Financials / Filter Important Financials1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filter financial statement data for important metrics  
  - *Input:* Raw financial data  
  - *Output:* Filtered financial data to merge nodes  
  - *Edge Cases:* Similar to above, data inconsistency or missing fields  

---

#### 2.3 Data Aggregation and Calculations

**Overview:**  
This block merges filtered data streams, performs calculations such as trailing twelve months (TTM), CAGR, and prediction models to prepare enriched data for AI analysis.

**Nodes Involved (Quarterly path):**  
- Merge1  
- Aggregate1  
- Calculate TTM  
- Merge  
- Aggregate  
- Calculate CAGR/Prediction Models  

**Nodes Involved (Annual path):**  
- Merge2  
- Aggregate2  
- Calculate CAGR/Prediction Models1  

**Node Details:**  

- **Merge1 / Merge2**  
  - *Type:* Merge  
  - *Role:* Combine filtered ratios, financials, and market cap or quotes into unified data sets  
  - *Inputs:* Filtered data nodes  
  - *Outputs:* Aggregation nodes  
  - *Edge Cases:* Merge conflicts or missing inputs  

- **Aggregate1 / Aggregate / Aggregate2**  
  - *Type:* Aggregate  
  - *Role:* Summarize or restructure merged data for computation  
  - *Inputs:* Merged data  
  - *Outputs:* Calculation code nodes  
  - *Edge Cases:* Aggregation errors if input data is malformed  

- **Calculate TTM**  
  - *Type:* Code (JavaScript)  
  - *Role:* Compute trailing twelve months financial metrics from quarterly data  
  - *Input:* Aggregated data  
  - *Output:* Merged node for final aggregation  
  - *Edge Cases:* Missing quarterly data, calculation errors  

- **Calculate CAGR/Prediction Models / Calculate CAGR/Prediction Models1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Calculate CAGR and generate basic prediction models for stock performance  
  - *Input:* Aggregated and processed financial data  
  - *Output:* Sent to AI analysis nodes  
  - *Edge Cases:* Mathematical errors, division by zero, missing historical data  

---

#### 2.4 AI Processing and Reporting

**Overview:**  
This block uses Google Gemini AI models via LangChain nodes to analyze the enriched data and produce natural language investment recommendations. The output is converted to HTML and file formats and saved locally for review or further use.

**Nodes Involved:**  
- Google Gemini Chat Model / Google Gemini Chat Model1  
- Executive Stock Analyst / Executive Stock Analyst (LangChain agents)  
- Convert to HTML / Convert to HTML2  
- Convert to Data / Convert to Data1  
- Save to Computer / Save to Computer1  

**Node Details:**  

- **Google Gemini Chat Model / Google Gemini Chat Model1**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* Provide AI language model interface for chat-based analysis  
  - *Input:* Calculated financial metrics and prediction data  
  - *Output:* Feeds into LangChain agent nodes  
  - *Edge Cases:* API quota, connection failures, model response delays  

- **Executive Stock Analyst (both instances)**  
  - *Type:* LangChain Agent  
  - *Role:* Advanced AI agent that generates detailed investment recommendations based on input data  
  - *Input:* Output from Gemini Chat Models and calculations  
  - *Output:* Human-readable analysis text to conversion nodes  
  - *Edge Cases:* Model hallucinations, incomplete data causing poor recommendations  

- **Convert to HTML / Convert to HTML2**  
  - *Type:* Code (JavaScript)  
  - *Role:* Transform AI-generated text into formatted HTML reports  
  - *Output:* Converted data nodes  
  - *Edge Cases:* Formatting errors if input text contains unexpected characters  

- **Convert to Data / Convert to Data1**  
  - *Type:* Convert to File  
  - *Role:* Convert HTML reports into files (e.g., PDF, HTML file) for saving  
  - *Output:* Saved files to disk nodes  
  - *Edge Cases:* File system permissions, disk space issues  

- **Save to Computer / Save to Computer1**  
  - *Type:* Read/Write File  
  - *Role:* Persist generated report files locally  
  - *Edge Cases:* File path errors, write permissions  

---

### 3. Summary Table

| Node Name                               | Node Type                                | Functional Role                                  | Input Node(s)                          | Output Node(s)                         | Sticky Note                          |
|----------------------------------------|-----------------------------------------|-------------------------------------------------|---------------------------------------|--------------------------------------|------------------------------------|
| Use if most recent company filing is quarterly report | Form Trigger                            | Entry point for quarterly report processing     | None                                  | Quote, Financials, Ratios, MarketCap |                                    |
| Use if most recent filing is annual report            | Form Trigger                            | Entry point for annual report processing        | None                                  | Quote1, Financials1, Ratios1          |                                    |
| Ratios                                 | HTTP Request                            | Fetch financial ratios                           | Use if most recent company filing ... | Filter Important Ratios                |                                    |
| Ratios1                                | HTTP Request                            | Fetch financial ratios (annual)                   | Use if most recent filing is annual...| Filter Important Ratios1               |                                    |
| Financials                             | HTTP Request                            | Fetch financial statements                       | Use if most recent company filing ... | Filter Important Financials            |                                    |
| Financials1                            | HTTP Request                            | Fetch financial statements (annual)               | Use if most recent filing is annual...| Filter Important Financials1           |                                    |
| MarketCap                             | HTTP Request                            | Fetch market capitalization (quarterly path)   | Use if most recent company filing ... | Merge1                               |                                    |
| Quote                                 | HTTP Request                            | Fetch current stock quote (quarterly path)      | Use if most recent company filing ... | Merge                                |                                    |
| Quote1                                | HTTP Request                            | Fetch current stock quote (annual path)          | Use if most recent filing is annual...| Merge2                               |                                    |
| Filter Important Ratios                | Code                                   | Filter key financial ratios (quarterly)         | Ratios                                | Merge1                               |                                    |
| Filter Important Ratios1               | Code                                   | Filter key financial ratios (annual)             | Ratios1                               | Merge2                               |                                    |
| Filter Important Financials            | Code                                   | Filter key financial metrics (quarterly)        | Financials                            | Merge                                |                                    |
| Filter Important Financials1           | Code                                   | Filter key financial metrics (annual)            | Financials1                           | Merge2                               |                                    |
| Merge1                                | Merge                                  | Combine filtered quarterly data                  | Filter Important Ratios, MarketCap, Filter Important Financials | Aggregate1                            |                                    |
| Merge                                 | Merge                                  | Combine quarterly final filtered data            | Calculate TTM, Filter Important Ratios, Filter Important Financials | Aggregate                             |                                    |
| Merge2                                | Merge                                  | Combine annual final filtered data                | Filter Important Ratios1, Filter Important Financials1, Quote1 | Aggregate2                            |                                    |
| Aggregate1                            | Aggregate                              | Aggregate quarterly merged data                  | Merge1                                | Calculate TTM                        |                                    |
| Aggregate                             | Aggregate                              | Aggregate quarterly merged data                  | Merge                                 | Calculate CAGR/Prediction Models     |                                    |
| Aggregate2                            | Aggregate                              | Aggregate annual merged data                      | Merge2                                | Calculate CAGR/Prediction Models1    |                                    |
| Calculate TTM                        | Code                                   | Calculate trailing twelve months data            | Aggregate1                           | Merge                               |                                    |
| Calculate CAGR/Prediction Models       | Code                                   | Calculate CAGR and generate prediction models    | Aggregate                            | Executive Stock Analyst              |                                    |
| Calculate CAGR/Prediction Models1      | Code                                   | Calculate CAGR and generate prediction models (annual) | Aggregate2                          | Executive Stock Analyst              |                                    |
| Google Gemini Chat Model              | LangChain Google Gemini Chat Model     | AI language model interface (quarterly)          | Calculate CAGR/Prediction Models      | Executive Stock Analyst (agent)     |                                    |
| Google Gemini Chat Model1             | LangChain Google Gemini Chat Model     | AI language model interface (annual)              | Calculate CAGR/Prediction Models1     | Executive Stock Analyst (agent)     |                                    |
| Executive Stock Analyst               | LangChain Agent                        | AI agent generating investment analysis (annual) | Google Gemini Chat Model1             | Convert to HTML2                    |                                    |
| Executive Stock Analyst               | LangChain Agent                        | AI agent generating investment analysis (quarterly) | Google Gemini Chat Model              | Convert to HTML                    |                                    |
| Convert to HTML                      | Code                                   | Convert AI output to HTML report (quarterly)     | Executive Stock Analyst               | Convert to Data                    |                                    |
| Convert to HTML2                     | Code                                   | Convert AI output to HTML report (annual)        | Executive Stock Analyst               | Convert to Data1                   |                                    |
| Convert to Data                      | Convert to File                       | Convert HTML to file format (quarterly)           | Convert to HTML                      | Save to Computer                  |                                    |
| Convert to Data1                     | Convert to File                       | Convert HTML to file format (annual)               | Convert to HTML2                     | Save to Computer1                 |                                    |
| Save to Computer                    | Read/Write File                     | Save file locally (quarterly)                       | Convert to Data                     | None                              |                                    |
| Save to Computer1                   | Read/Write File                     | Save file locally (annual)                           | Convert to Data1                    | None                              |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Nodes for Input Reception**  
   - Create two Form Trigger nodes named:  
     - "Use if most recent company filing is quarterly report"  
     - "Use if most recent filing is annual report"  
   - Configure each with unique webhook IDs to receive form submissions indicating the filing type.

2. **Set Up HTTP Request Nodes to Fetch Financial Data**  
   - For quarterly reports: Create nodes named "Ratios", "Financials", "MarketCap", "Quote". Configure each to fetch respective data from the financial API using appropriate endpoints, authentication, and parameters (e.g., stock symbol).
   - For annual reports: Create nodes "Ratios1", "Financials1", "Quote1" similarly configured for annual data.

3. **Create Code Nodes to Filter Important Data**  
   - For quarterly path: Create "Filter Important Ratios" and "Filter Important Financials" nodes as JavaScript code nodes to parse and filter desired metrics from raw API data.
   - For annual path: Similarly create "Filter Important Ratios1" and "Filter Important Financials1".

4. **Create Merge Nodes for Data Combination**  
   - "Merge1" to combine filtered quarterly data streams.  
   - "Merge2" to combine filtered annual data streams.  
   - "Merge" to combine quarterly calculated data and filtered ratios/financials.

5. **Create Aggregate Nodes**  
   - "Aggregate1" aggregates quarterly merged data.  
   - "Aggregate" aggregates quarterly merged data after TTM calculation.  
   - "Aggregate2" aggregates annual merged data.

6. **Create Calculation Code Nodes**  
   - "Calculate TTM": JavaScript code to compute trailing twelve months metrics from quarterly aggregates.  
   - "Calculate CAGR/Prediction Models": Calculate CAGR and generate prediction models for quarterly data.  
   - "Calculate CAGR/Prediction Models1": Same as above for annual data.

7. **Set Up Google Gemini AI Model Nodes**  
   - "Google Gemini Chat Model" and "Google Gemini Chat Model1": Configure with Google Gemini API credentials, setting prompt templates to accept computed data and request investment recommendation generation.

8. **Set Up LangChain Agent Nodes**  
   - "Executive Stock Analyst" nodes (two instances) configured to use outputs from Gemini Chat Models to perform advanced AI analysis.

9. **Create Code Nodes to Convert Output to HTML**  
   - "Convert to HTML" and "Convert to HTML2": JavaScript code nodes to format AI text output into HTML reports.

10. **Create Convert to File Nodes**  
    - "Convert to Data" and "Convert to Data1": Convert HTML reports into file formats suitable for saving.

11. **Create Read/Write File Nodes to Save Reports**  
    - "Save to Computer" and "Save to Computer1": Configure to write report files locally; set file paths and permissions accordingly.

12. **Connect Nodes According to the Workflow Logic**  
    - Connect form triggers to their respective HTTP request nodes.  
    - Connect HTTP requests to filter code nodes.  
    - Connect filters to merges, merges to aggregates, aggregates to calculations, calculations to AI models, AI models to conversion nodes, and finally to file saving nodes.

13. **Ensure Credentials Setup**  
    - Configure API credentials for financial data provider(s).  
    - Configure Google Gemini API credentials in LangChain nodes.

14. **Test each branch independently**  
    - Submit test inputs for both quarterly and annual report paths.  
    - Confirm data retrieval, processing, AI response generation, and report saving success.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow integrates Google Gemini AI models via LangChain for advanced natural language analysis. | See https://cloud.google.com/vertex-ai/docs/generative-ai for Google Gemini integration details.          |
| The financial data API endpoints and credentials are abstracted in HTTP Request nodes; ensure valid API keys and rate limits are respected. | Typically requires Finnhub or equivalent API access with proper authentication.                           |
| The workflow assumes local file system access for saving reports; adjust "Save to Computer" nodes for environment compatibility. | For cloud or container setups, adapt file storage accordingly.                                           |
| The two parallel branches cater specifically to quarterly vs annual filings, enabling tailored data processing for each case. | This design improves data accuracy based on filing type.                                                 |
| Use of JavaScript code nodes for filtering and calculations allows easy customization of financial metrics analyzed. | Adjust filtering logic to fit specific investment criteria or additional KPIs.                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a platform for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.