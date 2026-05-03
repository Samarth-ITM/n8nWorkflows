Predict and Forecast HDB Flat Prices with GPT-4o and Google Sheets Analytics

https://n8nworkflows.xyz/workflows/predict-and-forecast-hdb-flat-prices-with-gpt-4o-and-google-sheets-analytics-10891


# Predict and Forecast HDB Flat Prices with GPT-4o and Google Sheets Analytics

---

# Predict and Forecast HDB Flat Prices with GPT-4o and Google Sheets Analytics  
*Workflow Name:* GPT-4o HDB Flat Price Prediction and Forecasting System

---

### 1. Workflow Overview

This workflow automates the monthly collection, processing, analysis, and forecasting of Singapore’s HDB resale flat prices using historical and current-year data. It integrates advanced statistical techniques with AI-based forecasting (GPT-4o) and stores the results in Google Sheets for reporting and downstream use.

**Target Use Cases:**  
- Real estate market price forecasting with confidence intervals  
- Financial market trend analysis using multi-year historical data  
- Automated, recurrent data-driven insights for property investment and policy analysis  

**Logical Blocks:**  
- **1.1 Data Collection Layer:** Scheduled monthly trigger fetching current year and three years of historical HDB resale data from official API sources.  
- **1.2 Data Consolidation & Cleaning:** Merging multiple datasets, cleaning, normalizing, and validating data for quality and consistency.  
- **1.3 Pattern & Trend Analysis:** Statistical pattern mining and time series analysis to extract key features and trends.  
- **1.4 AI Forecasting Agent (Intelligence Hub):** Combines GPT-4o language model with custom statistical forecasting and calculator tools to generate probabilistic forecasts.  
- **1.5 Output & Storage:** Formats forecasts and saves them to a configured Google Sheets document for audit and accessibility.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Collection Layer

- **Overview:**  
  This block runs monthly, triggering API calls to retrieve HDB resale flat price data for the current year plus three preceding years in parallel. It ensures comprehensive historical context is available for forecasting.

- **Nodes Involved:**  
  - Monthly Data Collection Trigger  
  - Workflow Configuration  
  - Fetch Current Year HDB Data  
  - Fetch Historical Data Year 1  
  - Fetch Historical Data Year 2  
  - Fetch Historical Data Year 3  
  - Merge All Historical Data  

- **Node Details:**  

  - **Monthly Data Collection Trigger**  
    - *Type:* Schedule Trigger  
    - *Config:* Triggers once every month at 2 AM  
    - *Inputs:* None (start node)  
    - *Outputs:* To Workflow Configuration  
    - *Edge Cases:* Missed triggers if n8n is down; time zone considerations; ensure server time sync

  - **Workflow Configuration**  
    - *Type:* Set Node  
    - *Config:* Sets static parameters: API base URL, current year ("2024"), three historical years ("2023", "2022", "2021"), and a placeholder for the HDB data resource ID  
    - *Inputs:* From trigger  
    - *Outputs:* To four Fetch Data nodes in parallel  
    - *Edge Cases:* Placeholder resource ID must be replaced with actual API resource ID before running

  - **Fetch Current Year HDB Data**  
    - *Type:* HTTP Request  
    - *Config:* Queries the HDB API for the current year’s data filtered by month/year, limit 10,000 records, expects JSON  
    - *Inputs:* From Workflow Configuration  
    - *Outputs:* To Merge All Historical Data (input 0)  
    - *Edge Cases:* API rate limits, network timeouts, malformed responses, empty datasets

  - **Fetch Historical Data Year 1 / Year 2 / Year 3**  
    - *Type:* HTTP Request (x3)  
    - *Config:* Same as current year fetch, but for years 2023, 2022, 2021 respectively, with similar filters and limits  
    - *Inputs:* From Workflow Configuration  
    - *Outputs:* To Merge All Historical Data (inputs 1, 2, 3)  
    - *Edge Cases:* Same as current year fetch; ensure API supports historical queries; handle partial data availability

  - **Merge All Historical Data**  
    - *Type:* Merge Node  
    - *Config:* Merges four inputs into one consolidated dataset  
    - *Inputs:* From all four Fetch Data nodes  
    - *Outputs:* To Data Cleaning and Normalization  
    - *Edge Cases:* Duplicate or inconsistent records; input data format variations

---

#### 1.2 Data Consolidation & Cleaning

- **Overview:**  
  Processes the merged dataset by cleaning, normalizing field values, standardizing formats, removing duplicates, and validating data quality to ensure accurate downstream analysis.

- **Nodes Involved:**  
  - Data Cleaning and Normalization  

- **Node Details:**  

  - **Data Cleaning and Normalization**  
    - *Type:* Code Node (JavaScript)  
    - *Config:* Custom JS code that:  
      - Normalizes town names to uppercase trimmed strings  
      - Standardizes flat types (e.g., "1 ROOM" → "1-ROOM")  
      - Parses prices, floor areas, dates, and other fields, removing invalid or unrealistic values  
      - Removes duplicates using composite keys  
      - Validates presence of required fields (month, town, flat type, resale price)  
      - Filters out unrealistic resale prices (>2,000,000 SGD) and floor areas (20-300 sqm)  
      - Adds derived fields: year extracted from month, price per sqm  
    - *Inputs:* Merged dataset  
    - *Outputs:* To Statistical Pattern Mining and Time Series Analysis nodes  
    - *Edge Cases:* Empty or invalid input data; parsing errors; records filtered out entirely (workflow throws error if no valid data remains)  
    - *Notes:* Logs summary statistics on counts and duplicates

---

#### 1.3 Pattern & Trend Analysis

- **Overview:**  
  This block performs advanced statistical mining and time series analysis to extract key patterns, correlations, seasonal effects, and trend indicators from cleaned data.

- **Nodes Involved:**  
  - Statistical Pattern Mining  
  - Time Series Analysis  
  - Aggregate Statistical Features  

- **Node Details:**  

  - **Statistical Pattern Mining**  
    - *Type:* Code Node (JavaScript)  
    - *Config:*  
      - Groups data by flat type and town  
      - Calculates mean, median, std dev, quartiles for prices, floor levels, lease remaining, flat sizes  
      - Computes Pearson correlations between price and floor area, lease remaining, size  
      - Calculates year-over-year growth rates of average prices  
      - Detects seasonal patterns by monthly averages  
      - Returns comprehensive statistical insights and analysis timestamp  
    - *Inputs:* Cleaned data  
    - *Outputs:* To Aggregate Statistical Features  
    - *Edge Cases:* Empty datasets; missing fields; zero variance data causing correlations to be zero

  - **Time Series Analysis**  
    - *Type:* Code Node (JavaScript)  
    - *Config:*  
      - Sorts data by date  
      - Computes moving averages (3, 6, 12 months)  
      - Calculates exponential moving averages (EMA)  
      - Computes momentum indicators and rate of change (ROC)  
      - Identifies trend direction and strength  
      - Calculates volatility metrics (overall and rolling 12-month)  
      - Detects cyclical seasonal indices  
      - Summarizes data range and mean price  
    - *Inputs:* Cleaned data  
    - *Outputs:* To Aggregate Statistical Features  
    - *Edge Cases:* Insufficient data points for longer moving averages; missing or malformed dates

  - **Aggregate Statistical Features**  
    - *Type:* Aggregate Node  
    - *Config:* Aggregates inputs from Statistical Pattern Mining and Time Series Analysis into a single JSON object  
    - *Inputs:* From both pattern mining and time series nodes  
    - *Outputs:* To AI Forecasting Agent  
    - *Edge Cases:* Mismatched data structures; empty inputs

---

#### 1.4 AI Forecasting Agent (Intelligence Hub)

- **Overview:**  
  This block integrates GPT-4o with custom statistical forecasting and calculator tools in an AI agent node to generate probabilistic forecasts for the next 12 months, considering all extracted features and trends.

- **Nodes Involved:**  
  - AI Forecasting Agent  
  - OpenAI GPT-4 Model  
  - Statistical Forecasting Tool  
  - Calculator Tool  
  - Structured Forecast Output Parser  

- **Node Details:**  

  - **AI Forecasting Agent**  
    - *Type:* LangChain Agent Node  
    - *Config:*  
      - Uses GPT-4o for natural language reasoning on real estate forecasting  
      - Integrates Statistical Forecasting Tool and Calculator Tool for advanced calculations  
      - Receives aggregated statistical features as input  
      - Defines prompt instructing expert-level analysis considering seasonal, market, location factors  
      - Outputs structured forecast with confidence intervals  
    - *Inputs:* Aggregated statistical features  
    - *Outputs:* To Format Forecast Results  
    - *Edge Cases:* API quota limits; malformed AI responses; latency issues

  - **OpenAI GPT-4 Model**  
    - *Type:* LangChain Chat OpenAI Node  
    - *Config:* Model set to "gpt-4o" with linked OpenAI API credentials  
    - *Inputs:* AI Forecasting Agent (language model input)  
    - *Outputs:* AI Forecasting Agent (language model output)  
    - *Edge Cases:* Authentication errors; model unavailability; network issues

  - **Statistical Forecasting Tool**  
    - *Type:* LangChain Tool (Code)  
    - *Config:*  
      - Implements linear regression, exponential smoothing, Monte Carlo simulations  
      - Accepts historical price data JSON input  
      - Returns JSON forecast with 12-month predictions, confidence intervals, and model metrics  
    - *Inputs:* AI Forecasting Agent (tool input)  
    - *Outputs:* AI Forecasting Agent (tool output)  
    - *Edge Cases:* Invalid input JSON; insufficient data points; runtime exceptions in code

  - **Calculator Tool**  
    - *Type:* LangChain Calculator Tool  
    - *Config:* Provides calculator capabilities to AI agent for intermediate computations  
    - *Inputs:* AI Forecasting Agent (tool input)  
    - *Outputs:* AI Forecasting Agent (tool output)  
    - *Edge Cases:* Calculation errors or invalid expressions

  - **Structured Forecast Output Parser**  
    - *Type:* LangChain Output Parser Structured  
    - *Config:*  
      - Defines a strict JSON schema for forecast outputs including fields: flat_type, town, forecast_month, predicted_price, lower_bound, upper_bound, confidence_level (0-1 scale), trend_direction, key_factors  
      - Ensures AI output conforms to expected structured format  
    - *Inputs:* AI Forecasting Agent (output parser input)  
    - *Outputs:* AI Forecasting Agent (output parser output)  
    - *Edge Cases:* Parsing failures if AI output does not match schema

---

#### 1.5 Output & Storage

- **Overview:**  
  Formats the structured forecast results by adding metadata (forecast date, model version, data source) and appends or updates records in a configured Google Sheets document.

- **Nodes Involved:**  
  - Format Forecast Results  
  - Save to Google Sheets  

- **Node Details:**  

  - **Format Forecast Results**  
    - *Type:* Set Node  
    - *Config:* Adds fields:  
      - forecast_date (current ISO timestamp)  
      - model_version ("v1.0")  
      - data_source ("Singapore HDB Data.gov.sg")  
    - *Inputs:* AI Forecasting Agent output  
    - *Outputs:* Save to Google Sheets  
    - *Edge Cases:* Date formatting errors unlikely; ensure $now context is available

  - **Save to Google Sheets**  
    - *Type:* Google Sheets Node  
    - *Config:*  
      - Operation: Append or update rows based on matching columns: flat_type, town, forecast_month  
      - Maps forecast fields (predicted_price, bounds, trend_direction, confidence_level) and metadata to sheet columns  
      - Sheet name: "HDB Price Forecasts"  
      - Document ID: Placeholder to be replaced with actual Google Sheets document ID  
    - *Credentials:* Google Sheets OAuth2 credentials required and linked  
    - *Inputs:* Formatted forecast results  
    - *Edge Cases:* API limits, auth failures, sheet permission issues, schema mismatches

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                              | Input Node(s)                               | Output Node(s)                             | Sticky Note                                                                                          |
|------------------------------|----------------------------------|----------------------------------------------|---------------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|
| Monthly Data Collection Trigger | Schedule Trigger                 | Monthly trigger to start data collection    | None                                        | Workflow Configuration                     | ## 1. Data Collection Layer: Fetches current-year and 3 years of historical data monthly           |
| Workflow Configuration        | Set                              | Defines API URLs, years, resource ID        | Monthly Data Collection Trigger             | Fetch Current Year HDB Data, Fetch Historical Data Year 1/2/3 | Same as above                                                                                       |
| Fetch Current Year HDB Data   | HTTP Request                     | Fetch current year HDB resale flat data     | Workflow Configuration                      | Merge All Historical Data                  | Same as above                                                                                       |
| Fetch Historical Data Year 1  | HTTP Request                     | Fetch HDB resale data for year 2023          | Workflow Configuration                      | Merge All Historical Data                  | Same as above                                                                                       |
| Fetch Historical Data Year 2  | HTTP Request                     | Fetch HDB resale data for year 2022          | Workflow Configuration                      | Merge All Historical Data                  | Same as above                                                                                       |
| Fetch Historical Data Year 3  | HTTP Request                     | Fetch HDB resale data for year 2021          | Workflow Configuration                      | Merge All Historical Data                  | Same as above                                                                                       |
| Merge All Historical Data     | Merge                            | Consolidate all fetched datasets             | Fetch Current Year + Historical Data nodes | Data Cleaning and Normalization            | ## 2. Data Consolidation & Cleaning: Merges datasets, normalizes values                            |
| Data Cleaning and Normalization | Code                            | Data cleaning, normalization, validation     | Merge All Historical Data                   | Statistical Pattern Mining, Time Series Analysis | Same as above                                                                                       |
| Statistical Pattern Mining    | Code                            | Statistical analysis and pattern mining      | Data Cleaning and Normalization             | Aggregate Statistical Features             | ## 3. Pattern & Trend Analysis: Statistical mining and pattern extraction                          |
| Time Series Analysis          | Code                            | Time series feature extraction                | Data Cleaning and Normalization             | Aggregate Statistical Features             | Same as above                                                                                       |
| Aggregate Statistical Features | Aggregate                      | Aggregates statistical & time series features| Statistical Pattern Mining, Time Series Analysis | AI Forecasting Agent                       | Same as above                                                                                       |
| AI Forecasting Agent          | LangChain Agent                 | AI-based forecasting combining GPT-4o and tools | Aggregate Statistical Features            | Format Forecast Results                     | ## 4. AI Forecasting Agent: Synthesizes insights into predictions                                 |
| OpenAI GPT-4 Model            | LangChain Chat OpenAI           | Provides GPT-4o language model                | AI Forecasting Agent (languageModel input) | AI Forecasting Agent (languageModel output)| Same as above                                                                                       |
| Statistical Forecasting Tool  | LangChain Tool Code             | Performs statistical forecasting calculations | AI Forecasting Agent (tool input)           | AI Forecasting Agent (tool output)          | Same as above                                                                                       |
| Calculator Tool              | LangChain Tool Calculator        | Provides calculation support                   | AI Forecasting Agent (tool input)           | AI Forecasting Agent (tool output)          | Same as above                                                                                       |
| Structured Forecast Output Parser | LangChain Output Parser Structured | Parses AI forecast output into structured JSON | AI Forecasting Agent (outputParser input) | AI Forecasting Agent (outputParser output) | Same as above                                                                                       |
| Format Forecast Results       | Set                             | Adds metadata and formatting to forecasts    | AI Forecasting Agent                        | Save to Google Sheets                       | ## 5. Output & Storage: Formats and saves forecasts                                              |
| Save to Google Sheets         | Google Sheets                   | Append/update forecast data into spreadsheet | Format Forecast Results                     | None                                       | Same as above                                                                                       |
| Sticky Note (multiple nodes)  | Sticky Note                    | Informational notes describing workflow stages| N/A                                         | N/A                                        | Various notes on workflow design, setup, use cases, benefits (see notes section)                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: Monthly Data Collection Trigger  
   - Configure to run monthly at 2:00 AM server time

2. **Add a Set node**  
   - Name: Workflow Configuration  
   - Assign variables:  
     - `hdbApiBaseUrl` = "https://data.gov.sg/api/action/datastore_search"  
     - `currentYear` = "2024"  
     - `historicalYear1` = "2023"  
     - `historicalYear2` = "2022"  
     - `historicalYear3` = "2021"  
     - `resourceId` = "<Replace with actual HDB Resource ID>"  
   - Connect output from Monthly Data Collection Trigger

3. **Create four HTTP Request nodes** for data fetching:  
   - Names: Fetch Current Year HDB Data, Fetch Historical Data Year 1, Year 2, Year 3  
   - For each, configure URL as:  
     ```
     {{$node["Workflow Configuration"].json["hdbApiBaseUrl"]}}?resource_id={{$node["Workflow Configuration"].json["resourceId"]}}&filters={"month":"{{$node["Workflow Configuration"].json["<year_variable>"]}}"}&limit=10000
     ```  
   - Replace `<year_variable>` with `currentYear`, `historicalYear1`, `historicalYear2`, `historicalYear3` respectively  
   - Set response format to JSON  
   - Connect outputs from Workflow Configuration node

4. **Add a Merge node**  
   - Name: Merge All Historical Data  
   - Set inputs to 4 (for four HTTP Request nodes)  
   - Connect all four Fetch Data nodes outputs

5. **Add a Code node**  
   - Name: Data Cleaning and Normalization  
   - Paste the provided JavaScript cleaning and normalization code  
   - Connect output from Merge All Historical Data

6. **Add two Code nodes for analysis:**  
   - Statistical Pattern Mining (paste corresponding JS code)  
   - Time Series Analysis (paste corresponding JS code)  
   - Connect both nodes from Data Cleaning and Normalization node output

7. **Add an Aggregate node**  
   - Name: Aggregate Statistical Features  
   - Aggregate inputs from Statistical Pattern Mining and Time Series Analysis nodes  
   - Connect outputs accordingly

8. **Set up AI Forecasting Agent:**  
   - Add LangChain Agent node named AI Forecasting Agent  
   - Configure prompt text to instruct expert real estate forecasting using statistical features (as provided)  
   - Link AI Forecasting Agent with:  
     - OpenAI GPT-4 Chat node (model "gpt-4o") with OpenAI API credentials  
     - Statistical Forecasting Tool node (paste provided JS code for forecasting)  
     - Calculator Tool node (default calculator)  
     - Structured Forecast Output Parser node (define schema as provided)  
   - Connect Aggregate Statistical Features node output to AI Forecasting Agent input  
   - Connect OpenAI GPT-4, Statistical Forecasting Tool, Calculator, and Output Parser nodes as tools and language model references inside AI Forecasting Agent

9. **Add a Set node**  
   - Name: Format Forecast Results  
   - Add fields:  
     - `forecast_date` = current ISO timestamp (`{{$now.toISO()}}`)  
     - `model_version` = "v1.0"  
     - `data_source` = "Singapore HDB Data.gov.sg"  
   - Connect output from AI Forecasting Agent

10. **Add Google Sheets node:**  
    - Name: Save to Google Sheets  
    - Configure operation: Append or update rows  
    - Map columns: flat_type, town, forecast_month, predicted_price, lower_bound, upper_bound, confidence_level, trend_direction, forecast_date, model_version  
    - Set Sheet name: "HDB Price Forecasts"  
    - Set Document ID: Replace placeholder with actual Google Sheets document ID  
    - Provide Google Sheets OAuth2 credentials  
    - Connect output from Format Forecast Results

11. **Final connections:**  
    - Monthly Data Collection Trigger → Workflow Configuration  
    - Workflow Configuration → All four Fetch Data nodes  
    - All Fetch Data nodes → Merge All Historical Data  
    - Merge All Historical Data → Data Cleaning and Normalization  
    - Data Cleaning and Normalization → Statistical Pattern Mining and Time Series Analysis (parallel)  
    - Statistical Pattern Mining, Time Series Analysis → Aggregate Statistical Features  
    - Aggregate Statistical Features → AI Forecasting Agent  
    - AI Forecasting Agent → Format Forecast Results → Save to Google Sheets

12. **Validation & Testing:**  
    - Replace placeholder values (API resource ID, Google Sheet ID)  
    - Ensure API credentials for OpenAI and Google Sheets are valid and connected  
    - Test end-to-end with sample data, verify output in Google Sheets  
    - Monitor error logs for API failures or parsing issues

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow runs on a monthly schedule to automate data retrieval, cleaning, analysis, forecasting, and storage without manual intervention.                                    | Sticky Note5                                                                                            |
| Setup requires valid API credentials for OpenAI GPT-4 and Google Sheets OAuth2 along with the HDB data resource ID from data.gov.sg.                                           | Sticky Note6                                                                                            |
| Use cases extend beyond real estate to finance or any domain needing multi-year historical trend forecasting with confidence intervals.                                        | Sticky Note8                                                                                            |
| Customization is possible for different data sources (e.g., stock prices, sensor data) and adjustable analysis windows (2-5 years).                                           | Sticky Note9                                                                                            |
| The AI Forecasting Agent synthesizes statistical and time series features with GPT-4o’s natural language capabilities and custom forecasting tools for robust prediction.     | Sticky Note3 and Sticky Note4                                                                           |
| For more information on HDB data APIs, visit https://data.gov.sg/developer APIs.                                                                                              | External resource (not embedded in workflow)                                                            |
| The statistical forecasting tool uses linear regression, exponential smoothing, and Monte Carlo simulations for probabilistic confidence intervals on price predictions.       | Embedded in Statistical Forecasting Tool node                                                           |
| The workflow’s modular design supports easy extension or replacement of nodes for alternative data sources or AI models if needed.                                            | General design principle                                                                                 |

---

**Disclaimer:** The text provided exclusively derives from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---