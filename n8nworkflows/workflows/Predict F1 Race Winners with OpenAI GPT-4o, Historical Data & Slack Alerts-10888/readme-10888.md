Predict F1 Race Winners with OpenAI GPT-4o, Historical Data & Slack Alerts

https://n8nworkflows.xyz/workflows/predict-f1-race-winners-with-openai-gpt-4o--historical-data---slack-alerts-10888


# Predict F1 Race Winners with OpenAI GPT-4o, Historical Data & Slack Alerts

### 1. Workflow Overview

This workflow automates the prediction of Formula 1 race winners by integrating real-time and historical data sources with advanced AI analysis and communication channels. It is designed for daily execution, targeting sports analytics platforms, betting services, fantasy leagues, and F1 news outlets. The workflow is logically divided into the following blocks:

- **1.1 Scheduled Data Retrieval:** Automatically triggers daily data fetches from multiple F1 data APIs including current season details, standings, qualifying results, race schedules, and circuit information.
- **1.2 Data Aggregation and Processing:** Merges diverse F1 data sources into a unified dataset and computes advanced driver performance metrics using historical trends.
- **1.3 Historical Data Vectorization and Retrieval:** Converts multi-year historical race data into vector embeddings for semantic search and provides a tool to retrieve relevant historical information.
- **1.4 AI Prediction Agent:** Uses a LangChain GPT-4o agent enhanced with historical data, real-time news, weather data, and statistical tools to generate a race winner prediction with confidence scores and key analysis points.
- **1.5 Prediction Formatting and Confidence Evaluation:** Formats the AI output and evaluates the confidence score against a threshold to determine subsequent actions.
- **1.6 Alerting and Data Storage:** Sends Slack alerts for high-confidence predictions, stores predictions in PostgreSQL, and logs them in Google Sheets for tracking.
- **1.7 Statistical Analysis Tool:** Provides advanced statistical functions to support AI analyses, including correlation and win rate calculations.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Retrieval

- **Overview:** Initiates workflow daily at 8 AM to fetch multiple real-time F1 data sets.
- **Nodes Involved:** Schedule Trigger, Workflow Configuration, Fetch Current F1 Season Data, Fetch Driver Standings, Fetch Constructor Standings, Fetch Historical Race Results, Fetch Qualifying Results, Fetch Circuit Information.
- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Trigger  
    - *Config:* Fires daily at 8:00 AM  
    - *Outputs:* Workflow Configuration  
    - *Failure Cases:* Timezone mismatch, trigger misconfiguration.

  - **Workflow Configuration**  
    - *Type:* Set node  
    - *Config:* Sets URLs for Ergast API base, current season (2025), placeholder URLs for news and weather APIs, historicalYears=3, confidenceThreshold=0.75  
    - *Outputs:* Several HTTP Request nodes  
    - *Failure Cases:* Invalid URLs, missing API endpoints.

  - **Fetch Current F1 Season Data**  
    - *Type:* HTTP Request  
    - *Config:* GET request to Ergast API for current season races JSON  
    - *Output:* Race schedule and circuit info  
    - *Failures:* Network errors, API downtime.

  - **Fetch Driver Standings**  
    - *Type:* HTTP Request  
    - *Config:* GET driver standings for current season  
    - *Output:* Driver rankings and points  
    - *Failures:* Same as above.

  - **Fetch Constructor Standings**  
    - *Type:* HTTP Request  
    - *Config:* GET constructor standings for current season  
    - *Output:* Constructor rankings and points  
    - *Failures:* Same as above.

  - **Fetch Historical Race Results**  
    - *Type:* HTTP Request  
    - *Config:* GET previous season race results JSON (currentSeason - 1)  
    - *Output:* Historical race finishes  
    - *Failures:* Same as above.

  - **Fetch Qualifying Results**  
    - *Type:* HTTP Request  
    - *Config:* GET qualifying results for current season  
    - *Output:* Qualifying times and positions  
    - *Failures:* Same as above.

  - **Fetch Circuit Information**  
    - *Type:* HTTP Request  
    - *Config:* GET circuit details for current season  
    - *Output:* Circuit metadata  
    - *Failures:* Same as above.

---

#### 1.2 Data Aggregation and Processing

- **Overview:** Combines all fetched F1 data sources into a single structured JSON and computes complex driver performance metrics.
- **Nodes Involved:** Merge F1 Data, Aggregate Historical Data, Calculate Driver Performance Metrics.
- **Node Details:**

  - **Merge F1 Data**  
    - *Type:* Code node  
    - *Config:* Receives multiple HTTP request outputs, identifies data type by structure, merges season, driver standings, and constructor standings into one object with timestamp  
    - *Key Logic:* Checks MRData structure to separate data  
    - *Output:* Unified F1 data JSON  
    - *Failures:* Unexpected data structures, missing keys.

  - **Aggregate Historical Data**  
    - *Type:* Aggregate node  
    - *Config:* Aggregates all historical data inputs into a single collection for processing  
    - *Output:* Aggregated data array  
    - *Failures:* Empty input arrays.

  - **Calculate Driver Performance Metrics**  
    - *Type:* Code node  
    - *Config:* Processes historical race results to calculate metrics per driver such as: podium rate, win rate, DNF rate, average finishing position, qualifying position, consistency score, recent form average, points per race  
    - *Key Expressions:* Iterates race results, computes statistics, sorts by total points  
    - *Output:* Array of driver metrics objects  
    - *Failures:* Missing or malformed race data, division by zero, null values.

---

#### 1.3 Historical Data Vectorization and Retrieval

- **Overview:** Converts historical F1 data into vector embeddings for semantic search and provides a retrieval tool for the AI agent.
- **Nodes Involved:** Document Loader, Recursive Text Splitter, OpenAI Embeddings1, Historical Data Vector Store, OpenAI Embeddings, Historical Data Retrieval Tool.
- **Node Details:**

  - **Document Loader**  
    - *Type:* LangChain Document Loader  
    - *Config:* Loads raw historical data text for processing  
    - *Output:* Text documents for splitting  
    - *Failures:* Empty or malformed input.

  - **Recursive Text Splitter**  
    - *Type:* LangChain Text Splitter  
    - *Config:* Splits documents into smaller contextual chunks for embedding  
    - *Output:* Text chunks  
    - *Failures:* Very large documents may cause performance issues.

  - **OpenAI Embeddings1**  
    - *Type:* LangChain Embeddings (OpenAI)  
    - *Config:* Converts text chunks into vector embeddings  
    - *Credentials:* OpenAI API key  
    - *Failures:* API rate limits, auth errors.

  - **Historical Data Vector Store**  
    - *Type:* LangChain In-Memory Vector Store  
    - *Config:* Inserts embeddings into vector memory for future retrieval  
    - *Output:* Stores under key `f1_historical_data`  
    - *Failures:* Memory limitations.

  - **OpenAI Embeddings**  
    - *Type:* LangChain Embeddings (OpenAI)  
    - *Config:* Generates embeddings for query retrieval tool  
    - *Credentials:* OpenAI API key  
    - *Failures:* Same as above.

  - **Historical Data Retrieval Tool**  
    - *Type:* LangChain Vector Store (retrieve-as-tool)  
    - *Config:* Tool for semantic search on historical data with top 10 closest results  
    - *Output:* Provides relevant historical context for AI agent  
    - *Failures:* Empty vector store, poor retrieval relevance.

---

#### 1.4 AI Prediction Agent

- **Overview:** Uses a GPT-4o powered LangChain agent that synthesizes all data, news, weather, and statistical analysis to predict the next F1 race winner with explanation.
- **Nodes Involved:** F1 Prediction Agent, Real-Time F1 News Tool, Weather Data Tool, Statistical Analysis Tool, OpenAI Chat Model.
- **Node Details:**

  - **F1 Prediction Agent**  
    - *Type:* LangChain Agent  
    - *Config:* Prompted as expert F1 analyst with access to historical data search, real-time news, weather, and statistics. Outputs winner name, confidence score (0-1), top 3 finishers, key factors, risk factors, and weather impact.  
    - *Inputs:* Merged data, retrieved historical data, news, weather, statistical tool  
    - *Output:* Detailed prediction JSON  
    - *Failures:* Prompt misconfiguration, tool integration errors.

  - **Real-Time F1 News Tool**  
    - *Type:* HTTP Request Tool  
    - *Config:* Fetches latest news on F1 (injuries, updates, race developments) from configured news API URL  
    - *Output:* News data for AI context  
    - *Failures:* API endpoint missing or invalid.

  - **Weather Data Tool**  
    - *Type:* HTTP Request Tool  
    - *Config:* Fetches weather forecast for upcoming race location (temperature, precipitation, wind)  
    - *Output:* Weather data for AI context  
    - *Failures:* API endpoint missing or invalid.

  - **Statistical Analysis Tool**  
    - *Type:* Code Tool  
    - *Config:* Provides functions for mean, median, standard deviation, win rates, correlation for AI to perform statistical reasoning  
    - *Output:* JSON-formatted statistical results  
    - *Failures:* Malformed input data.

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Config:* Uses GPT-4o model with temperature 0.3 for deterministic output  
    - *Credentials:* OpenAI API key  
    - *Output:* Generates natural language synthesis from agent prompt  
    - *Failures:* API rate limits, auth errors.

---

#### 1.5 Prediction Formatting and Confidence Evaluation

- **Overview:** Extracts and formats AI agent output, then evaluates confidence score to decide downstream actions.
- **Nodes Involved:** Format Prediction Output, Check Prediction Confidence.
- **Node Details:**

  - **Format Prediction Output**  
    - *Type:* Set node  
    - *Config:* Sets fields including predictionDate (current ISO timestamp), predictionSource ("AI Agent Analysis"), dataVersion ("v1.0"), confidenceScore, predictedWinner (both extracted from AI output), and analysisDepth ("comprehensive")  
    - *Output:* Formatted prediction JSON  
    - *Failures:* Incorrect JSON path extraction if AI output structure changes.

  - **Check Prediction Confidence**  
    - *Type:* If node  
    - *Config:* Checks if confidenceScore ≥ configured threshold (0.75 by default)  
    - *Output:* True branch triggers alert and database storage, false branch triggers logging only  
    - *Failures:* Data type mismatch, missing confidenceScore.

---

#### 1.6 Alerting and Data Storage

- **Overview:** Sends Slack alerts for high-confidence predictions, stores them in PostgreSQL, and logs in Google Sheets.
- **Nodes Involved:** Send High Confidence Alert, Store Prediction in Database, Log to Prediction Tracker.
- **Node Details:**

  - **Send High Confidence Alert**  
    - *Type:* Slack node  
    - *Config:* Sends formatted message with predicted winner, confidence %, key factors, and timestamp to configured Slack channel (requires OAuth2)  
    - *Credentials:* Slack OAuth2 account  
    - *Failures:* Slack API errors, invalid channel ID, auth issues.

  - **Store Prediction in Database**  
    - *Type:* Postgres node  
    - *Config:* Inserts prediction data into configured table with columns: prediction_date, predicted_winner, confidence_score, prediction_source, data_version, full_analysis  
    - *Failures:* DB connection errors, schema mismatch, missing permissions.

  - **Log to Prediction Tracker**  
    - *Type:* Google Sheets node  
    - *Config:* Appends or updates a row in specified Google Sheets document and sheet with Date, Data Version, Analysis Source, Confidence Score, Predicted Winner  
    - *Credentials:* Google Sheets OAuth2 account  
    - *Failures:* Auth errors, invalid document ID or sheet name.

---

#### 1.7 Statistical Analysis Tool

- **Overview:** Provides statistical calculations used by the AI agent to interpret F1 data trends and correlations.
- **Nodes Involved:** Statistical Analysis Tool.
- **Node Details:**

  - **Statistical Analysis Tool**  
    - *Type:* LangChain Tool (Code node)  
    - *Config:* Implements mean, median, standard deviation, win rate, correlation functions in JavaScript  
    - *Input:* JSON query with datasets to analyze  
    - *Output:* JSON string with analysis results including driver stats and correlations  
    - *Failures:* Invalid input format, math errors on empty datasets.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                                   | Input Node(s)                             | Output Node(s)                           | Sticky Note                                                                                      |
|-------------------------------|-----------------------------------------|-------------------------------------------------|------------------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Trigger                                 | Initiates workflow daily                         | None                                     | Workflow Configuration                   | ## How It Works: Daily 8 AM trigger for data retrieval                                          |
| Workflow Configuration       | Set                                     | Defines API URLs, parameters                     | Schedule Trigger                         | Fetch Current F1 Season Data, Fetch Driver Standings, Fetch Constructor Standings, Fetch Historical Race Results, Fetch Qualifying Results, Fetch Circuit Information | ## Setup Steps: Update API URLs, DB schema, Slack, Google Sheets                                |
| Fetch Current F1 Season Data | HTTP Request                            | Retrieves current season race data               | Workflow Configuration                   | Merge F1 Data                           | ## Data Collection: Fresh multi-source F1 season data                                          |
| Fetch Driver Standings       | HTTP Request                            | Retrieves driver standings                        | Workflow Configuration                   | Merge F1 Data                           | ## Data Collection                                                                             |
| Fetch Constructor Standings  | HTTP Request                            | Retrieves constructor standings                   | Workflow Configuration                   | Merge F1 Data                           | ## Data Collection                                                                             |
| Fetch Historical Race Results| HTTP Request                            | Retrieves last season race results                | Workflow Configuration                   | Aggregate Historical Data               | ## Data Collection                                                                             |
| Fetch Qualifying Results     | HTTP Request                            | Retrieves current season qualifying results       | Workflow Configuration                   | Aggregate Historical Data               | ## Data Collection                                                                             |
| Fetch Circuit Information    | HTTP Request                            | Retrieves circuit information                      | Workflow Configuration                   | Aggregate Historical Data               | ## Data Collection                                                                             |
| Merge F1 Data               | Code                                    | Combines multiple F1 data sources                  | Fetch Current F1 Season Data, Fetch Driver Standings, Fetch Constructor Standings | F1 Prediction Agent                    | ## Data Processing: Merges data for AI agent                                                  |
| Aggregate Historical Data    | Aggregate                               | Aggregates historical data inputs                  | Fetch Historical Race Results, Fetch Qualifying Results, Fetch Circuit Information | Calculate Driver Performance Metrics   | ## Data Processing                                                                           |
| Calculate Driver Performance Metrics | Code                           | Computes advanced driver metrics                    | Aggregate Historical Data               | Historical Data Vector Store            | ## Driver Performance Metrics: Calculates 8 key metrics                                       |
| Document Loader             | LangChain Document Loader               | Loads raw historical data text                      | Recursive Text Splitter                  | Historical Data Vector Store            | ## Historical Data Vectorization Pipeline                                                   |
| Recursive Text Splitter     | LangChain Text Splitter                 | Splits long texts into chunks                       | Document Loader                         | Document Loader                        | ## Historical Data Vectorization Pipeline                                                   |
| OpenAI Embeddings1          | LangChain Embeddings (OpenAI)           | Converts text chunks into embeddings                | Document Loader                         | Historical Data Vector Store            | ## Historical Data Vectorization Pipeline                                                   |
| Historical Data Vector Store| LangChain Vector Store (In-Memory)      | Stores vector embeddings of historical data         | OpenAI Embeddings1                      | Historical Data Retrieval Tool          | ## Historical Data Vectorization Pipeline                                                   |
| OpenAI Embeddings           | LangChain Embeddings (OpenAI)           | Embeddings for retrieval tool                        | F1 Prediction Agent                    | Historical Data Retrieval Tool          | ## Historical Data Vectorization Pipeline                                                   |
| Historical Data Retrieval Tool | LangChain Vector Store (retrieve-as-tool) | Provides semantic search on past data              | OpenAI Embeddings                      | F1 Prediction Agent                    | ## Historical Data Vectorization Pipeline                                                   |
| F1 Prediction Agent         | LangChain Agent                        | Core AI prediction logic with data & tools          | Merge F1 Data, Historical Data Retrieval Tool, Real-Time F1 News Tool, Weather Data Tool, Statistical Analysis Tool, OpenAI Chat Model | Format Prediction Output               | ## AI Prediction: GPT-4o analyzes all inputs                                                |
| Real-Time F1 News Tool      | HTTP Request Tool                      | Fetches latest F1 news                              | Workflow Configuration                   | F1 Prediction Agent                    | ## AI Prediction                                                                             |
| Weather Data Tool           | HTTP Request Tool                      | Fetches weather forecast for the upcoming race     | Workflow Configuration                   | F1 Prediction Agent                    | ## AI Prediction                                                                             |
| Statistical Analysis Tool   | LangChain Code Tool                    | Performs statistical calculations for AI            | F1 Prediction Agent                    | F1 Prediction Agent                    | ## AI Prediction                                                                             |
| OpenAI Chat Model           | LangChain LM Chat OpenAI GPT-4o         | Generates natural language output                    | F1 Prediction Agent                    | F1 Prediction Agent                    | ## AI Prediction                                                                             |
| Format Prediction Output    | Set                                   | Formats AI output with metadata                      | F1 Prediction Agent                    | Check Prediction Confidence            | ## Confidence Validation: Formats and prepares prediction                                   |
| Check Prediction Confidence | If                                    | Compares confidence score to threshold              | Format Prediction Output               | Send High Confidence Alert, Store Prediction in Database, Log to Prediction Tracker | ## Confidence Validation: Threshold gating for actions                                       |
| Send High Confidence Alert  | Slack                                 | Sends Slack alert on high-confidence predictions    | Check Prediction Confidence (true)     | None                                   | ## Alerting and Storage: Slack alert for confident predictions                              |
| Store Prediction in Database| Postgres                              | Stores prediction data in configured DB              | Check Prediction Confidence (true)     | None                                   | ## Alerting and Storage: Database record for predictions                                   |
| Log to Prediction Tracker   | Google Sheets                        | Logs prediction data to Google Sheets                | Check Prediction Confidence (false)    | None                                   | ## Alerting and Storage: Logs all predictions for historical tracking                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 08:00 local time.

2. **Add a Set Node for Workflow Configuration**  
   - Set variables:  
     - `f1ApiBaseUrl`: "https://ergast.com/api/f1"  
     - `currentSeason`: "2025"  
     - `newsApiUrl`: Placeholder for news API endpoint  
     - `weatherApiUrl`: Placeholder for weather API endpoint  
     - `historicalYears`: 3  
     - `confidenceThreshold`: 0.75

3. **Create HTTP Request Nodes to Fetch Data**  
   - Fetch Current Season Data: URL `{{ $json.f1ApiBaseUrl }}/{{ $json.currentSeason }}.json`  
   - Fetch Driver Standings: URL `{{ $json.f1ApiBaseUrl }}/{{ $json.currentSeason }}/driverStandings.json`  
   - Fetch Constructor Standings: URL `{{ $json.f1ApiBaseUrl }}/{{ $json.currentSeason }}/constructorStandings.json`  
   - Fetch Historical Race Results: URL `{{ $json.f1ApiBaseUrl }}/{{ Number($json.currentSeason) - 1 }}/results.json`  
   - Fetch Qualifying Results: URL `{{ $json.f1ApiBaseUrl }}/{{ $json.currentSeason }}/qualifying.json`  
   - Fetch Circuit Information: URL `{{ $json.f1ApiBaseUrl }}/{{ $json.currentSeason }}/circuits.json`

4. **Create a Code Node named "Merge F1 Data"**  
   - Paste JavaScript code to merge data from season, driver standings, and constructor standings into one JSON object with timestamp.

5. **Create an Aggregate Node named "Aggregate Historical Data"**  
   - Set to aggregate all historical data from qualifying, circuits, and historical race results.

6. **Add a Code Node "Calculate Driver Performance Metrics"**  
   - Paste the detailed JS code to compute driver metrics: podium rate, win rate, DNF rate, average finishing and qualifying positions, consistency, recent form, points per race.

7. **Set up LangChain Nodes for Document Processing**  
   - Add Recursive Text Splitter to chunk historical data text.  
   - Add Document Loader to ingest raw historical data.  
   - Add OpenAI Embeddings nodes (two instances) for embedding generation.  
   - Add Historical Data Vector Store (in-memory) node to store embeddings.  
   - Add Historical Data Retrieval Tool node to enable semantic search over embeddings.

8. **Set up HTTP Request Tool Nodes**  
   - Real-Time F1 News Tool: URL from workflow config `newsApiUrl`  
   - Weather Data Tool: URL from workflow config `weatherApiUrl`

9. **Add Statistical Analysis Tool**  
   - Create a LangChain Code Tool node with provided JS statistical functions.

10. **Create F1 Prediction Agent Node**  
    - Use LangChain Agent node.  
    - Configure prompt as expert F1 analyst.  
    - Attach tools: Historical Data Retrieval Tool, Real-Time News Tool, Weather Data Tool, Statistical Analysis Tool.  
    - Attach OpenAI Chat Model node: GPT-4o with temperature 0.3 and OpenAI credentials.

11. **Add a Set Node "Format Prediction Output"**  
    - Assign fields with prediction date, source, data version, confidence score, predicted winner (all from agent output).

12. **Add an If Node "Check Prediction Confidence"**  
    - Condition: confidenceScore ≥ confidenceThreshold (0.75).

13. **Create Slack Node "Send High Confidence Alert"**  
    - Configure Slack OAuth2 credentials.  
    - Set channel ID for alerts.  
    - Compose message with predicted winner, confidence score, key factors, and timestamp.

14. **Add Postgres Node "Store Prediction in Database"**  
    - Connect to your Postgres DB.  
    - Configure insert into predictions table with appropriate columns.

15. **Add Google Sheets Node "Log to Prediction Tracker"**  
    - Connect Google Sheets OAuth2 credentials.  
    - Configure appendOrUpdate operation with relevant columns.  
    - Specify document ID and sheet name.

16. **Connect Nodes According to Data Flow:**  
    - Schedule Trigger → Workflow Configuration → All HTTP Requests → Merge F1 Data → F1 Prediction Agent → Format Prediction Output → Check Prediction Confidence.  
    - True branch: Send High Confidence Alert → Store Prediction in Database.  
    - False branch: Log to Prediction Tracker.

17. **Test Workflow:**  
    - Verify API endpoints, credentials (OpenAI, Slack, Postgres, Google Sheets).  
    - Test each node independently to ensure data correctness.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| ## How It Works: Daily at 8 AM, merges latest F1 data and computes metrics, AI predicts winner with confidence. Slack alerts and stores results on high confidence.                                                               | Sticky Note on workflow purpose                                                                 |
| ## Setup Steps: Update placeholders for newsApiUrl, weatherApiUrl, DB table, Slack channel ID, and Google Sheets document ID. Test Ergast API connectivity (no auth needed).                                                        | Sticky Note on initial configuration                                                            |
| ## Use Cases: Suitable for sports analytics dashboards, betting platforms, F1 news sites, and fantasy F1 leagues. Provides daily race predictions with confidence scores.                                                        | Sticky Note on application contexts                                                             |
| ## Customization: Extend AI prompt to include constructor predictions. Swap Slack for Discord or Microsoft Teams for alerts.                                                                                                   | Sticky Note on extensibility                                                                     |
| ## Benefits: Saves time by automating data collection and analysis. Improves accuracy by combining multiple metrics and historical patterns.                                                                                   | Sticky Note on workflow advantages                                                             |
| ## Data Collection: Uses Ergast API for live standings, historical results, qualifying data, and circuits. Vital for freshness and multi-source insights.                                                                       | Sticky Note emphasizing data importance                                                        |
| ## Data Processing: Merges data and calculates eight driver metrics such as podium rate and consistency. These metrics uncover patterns not visible in raw standings.                                                           | Sticky Note emphasizing data processing                                                         |
| ## AI Prediction: LangChain GPT-4o agent synthesizes data, news, weather, and statistics to produce data-driven winner predictions citing specific stats.                                                                       | Sticky Note on AI capabilities                                                                  |
| ## Confidence Validation: Uses a 0.75 threshold to reduce alert fatigue and ensure actionable insights.                                                                                                                         | Sticky Note on confidence filtering                                                             |
| ## Driver Performance Metrics: Calculates advanced metrics capturing true driver performance across circuits and conditions, beyond raw points and positions.                                                                  | Sticky Note detailing metric calculation                                                        |
| ## Historical Data Vectorization Pipeline: Transforms 3 years of historical F1 data into searchable embeddings using OpenAI. Enables semantic search to find similar past race scenarios, critical for accurate AI predictions. | Sticky Note on vectorization and semantic search                                               |

---

**Disclaimer:** The provided content originates exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal or offensive material. All processed data is legal and publicly accessible.