Flight Deal Analyzer with Weather Data using GPT, Google Flights & WordPress

https://n8nworkflows.xyz/workflows/flight-deal-analyzer-with-weather-data-using-gpt--google-flights---wordpress-10621


# Flight Deal Analyzer with Weather Data using GPT, Google Flights & WordPress

---

### 1. Workflow Overview

This workflow automates the process of discovering and analyzing flight deals from user input, enriched with real-time weather data, and publishes insightful travel content to WordPress. It targets travel bloggers, deal hunters, and travel planners who want to streamline flight price research combined with weather-informed recommendations.

The workflow is logically divided into these blocks:

- **1.1 User Input Reception:** Captures travel preferences via a web form.
- **1.2 Flight Price Scraping & Extraction:** Queries Google Flights for live flight pricing data and extracts relevant details.
- **1.3 Weather Data Retrieval & Parsing:** Fetches current and forecast weather data for key Japanese cities.
- **1.4 AI Input Preparation & Enrichment:** Consolidates flight, user, and weather data for AI analysis.
- **1.5 AI Flight & Weather Analysis:** Uses OpenAI-powered chat and agent nodes to analyze combined data for travel insights.
- **1.6 Output Formatting & Distribution:** Formats AI results, posts to WordPress, sends Slack notifications, and responds to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception

- **Overview:** Collects user travel preferences, including departure city, preferred travel month, budget, and date flexibility via an embedded web form.
- **Nodes Involved:**  
  - User Input Form  
  - Extract Form Data

- **Node Details:**

  - **User Input Form**  
    - Type: Form Trigger  
    - Role: Provides a webhook-based form for users to submit travel preferences.  
    - Configuration: Path set to "japan-flight-analyzer", form titled "Japan Flight Price Analyzer" with fields for Departure City (required text), Preferred Month (required text), Maximum Budget (required number), and Flexible Dates (dropdown with Yes/No). The response mode is configured to feed data into the workflow.  
    - Inputs: HTTP webhook trigger (external user).  
    - Outputs: JSON with submitted form data.  
    - Edge Cases: Invalid or missing required fields; user abandonment.  
    - Version: 2.1

  - **Extract Form Data**  
    - Type: Set Node  
    - Role: Normalizes and extracts specific form data fields into named variables for downstream use.  
    - Configuration: Extracts `departureCity`, `preferredMonth`, `maxBudget` (number), and `flexibleDates` from the form JSON.  
    - Inputs: Output from User Input Form.  
    - Outputs: Clean JSON fields for later nodes.  
    - Edge Cases: Missing or malformed data fields, incorrect types.  
    - Version: 3.4

---

#### 2.2 Flight Price Scraping & Extraction

- **Overview:** Scrapes Google Flights for flight options based on user input; extracts flight pricing and details from HTML content.
- **Nodes Involved:**  
  - Scrape Flight Prices  
  - Extract Flight Data

- **Node Details:**

  - **Scrape Flight Prices**  
    - Type: HTTP Request  
    - Role: Sends a GET request to Google Flights search URL with query parameters dynamically composed from the departure city to Tokyo, Japan.  
    - Configuration: URL is https://www.google.com/travel/flights with query parameter `q=flights to Tokyo Japan from {{ departureCity }}`.  
    - Inputs: Extract Form Data output.  
    - Outputs: Raw HTML or JSON response body.  
    - Edge Cases: Request failures, Google blocking scraping, changes in Google Flights page layout, network timeouts.  
    - Version: 4.2

  - **Extract Flight Data**  
    - Type: HTML Extract  
    - Role: Parses the HTTP response HTML to extract flight details.  
    - Configuration: Uses HTML extraction operation; specifics of selectors are not detailed but presumably target flight prices and details.  
    - Inputs: Scrape Flight Prices output.  
    - Outputs: Cleaned flight data in structured format.  
    - Edge Cases: Selector failures due to page structure changes, empty or malformed HTML.  
    - Version: 1.2

---

#### 2.3 Weather Data Retrieval & Parsing

- **Overview:** Fetches current and 7-day forecast weather data for Tokyo, Osaka, and Sapporo to enrich flight deal analysis.
- **Nodes Involved:**  
  - Fetch Weather Data  
  - Parse Weather Data

- **Node Details:**

  - **Fetch Weather Data**  
    - Type: HTTP Request  
    - Role: Queries open-meteo.com API for weather data of three cities in Japan using latitude and longitude coordinates.  
    - Configuration:  
      - Latitude: "35.6762,34.6937,43.0642" (Tokyo, Osaka, Sapporo)  
      - Longitude: "139.6503,135.5023,141.3469"  
      - Current weather parameters: temperature_2m, weather_code, wind_speed_10m  
      - Daily forecast parameters: max/min temperature, precipitation sum, weather code  
      - Forecast days: 7  
      - Timezone: Asia/Tokyo  
    - Inputs: Triggered after AI Input preparation.  
    - Outputs: JSON weather data for multiple locations.  
    - Edge Cases: API rate limits, network errors, incomplete data, invalid parameters.  
    - Version: 4.2

  - **Parse Weather Data**  
    - Type: Set Node  
    - Role: Maps and structures the weather API response into separate objects for each city and forecast data.  
    - Configuration: Assigns `tokyoWeather`, `osakaWeather`, and `sapporoWeather` from the `current` array elements; assigns `weatherForecast` from the `daily` field.  
    - Inputs: Fetch Weather Data output.  
    - Outputs: Structured weather data JSON objects.  
    - Edge Cases: Missing indices in arrays, malformed JSON.  
    - Version: 3.4

---

#### 2.4 AI Input Preparation & Enrichment

- **Overview:** Combines flight data, user preferences, and parsed weather data into a single formatted string for AI analysis.
- **Nodes Involved:**  
  - Prepare AI Input  
  - Enrich AI Input with Weather

- **Node Details:**

  - **Prepare AI Input**  
    - Type: Set Node  
    - Role: Creates initial data fields `flightData` (raw extracted flight info) and `userPreferences` (formatted string summarizing user inputs).  
    - Configuration: Uses expressions to inject flight data and user preferences from prior nodes.  
    - Inputs: Extract Flight Data output.  
    - Outputs: JSON with `flightData` and `userPreferences`.  
    - Edge Cases: Missing flight data, expression errors.  
    - Version: 3.4

  - **Enrich AI Input with Weather**  
    - Type: Set Node  
    - Role: Concatenates flight data, user preferences, and detailed weather info into a comprehensive `combinedData` string for AI consumption.  
    - Configuration: Constructs multiline string with embedded temperature and wind speed data for each city and a JSON string of 7-day forecast.  
    - Inputs: Parse Weather Data output combined with Prepare AI Input data.  
    - Outputs: Single `combinedData` string field.  
    - Edge Cases: JSON serialization failures, missing weather data fields.  
    - Version: 3.4

---

#### 2.5 AI Flight & Weather Analysis

- **Overview:** Uses OpenAI-powered chat and agent nodes to analyze combined flight and weather data, producing structured travel insights and recommendations.
- **Nodes Involved:**  
  - AI Flight Analyzer  
  - Structured Output Parser  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Flight Analyzer**  
    - Type: LangChain Agent Node  
    - Role: Acts as an expert travel data analyst processing the combined input to generate a detailed analysis including cheapest flights, travel period recommendations, budget evaluation, weather-based travel tips, and alternative routes.  
    - Configuration:  
      - System message frames the AI as an expert travel analyst.  
      - Prompt includes combined data injection.  
      - Output parser enabled for structured response parsing.  
    - Inputs: Combined data from Enrich AI Input with Weather; AI language model injected via OpenAI Chat Model node.  
    - Outputs: Structured AI analysis output.  
    - Credentials: OpenAI API key configured.  
    - Edge Cases: API rate limits, prompt injection errors, parsing failures, model downtime.  
    - Version: 1.7

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI raw textual output into structured JSON for easier downstream processing and formatting.  
    - Configuration: Uses default structured output parsing.  
    - Inputs: AI Flight Analyzer output parser input port.  
    - Outputs: Parsed structured JSON.  
    - Edge Cases: Parsing errors if AI output does not conform to expected format.  
    - Version: 1.2

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the underlying GPT model ("gpt-realtime") for AI Flight Analyzer agent.  
    - Configuration: Model "gpt-realtime" selected; no additional options.  
    - Credentials: OpenAI API credentials linked.  
    - Inputs: Receives prompt from AI Flight Analyzer.  
    - Outputs: AI-generated text responses.  
    - Edge Cases: API errors, quota exhaustion.  
    - Version: 1

---

#### 2.6 Output Formatting & Distribution

- **Overview:** Prepares the AI analysis for user consumption, publishes results to WordPress, notifies Slack channel, and responds to the form submitter.
- **Nodes Involved:**  
  - Format Analysis Results  
  - Send to Slack  
  - Publish to WordPress  
  - Respond to User

- **Node Details:**

  - **Format Analysis Results**  
    - Type: Set Node  
    - Role: Consolidates AI analysis output, user email (timestamp), and weather data into a simplified JSON payload for notifications and posts.  
    - Configuration: Assigns `aiAnalysis` from AI output, `userEmail` from User Input Form submission timestamp, and `weatherData` from parsed weather info.  
    - Inputs: AI Flight Analyzer output.  
    - Outputs: Formatted JSON for downstream nodes.  
    - Edge Cases: Missing AI output, misaligned references.  
    - Version: 3.4

  - **Send to Slack**  
    - Type: Slack Node  
    - Role: Sends a formatted message summarizing flight search criteria, current weather, and AI analysis to a designated Slack channel.  
    - Configuration: Uses template expressions to dynamically fill message content including departure city, month, budget, weather for Tokyo, Osaka, Sapporo, and AI analysis summary.  
    - Inputs: Format Analysis Results output.  
    - Outputs: Slack message delivery status.  
    - Credentials: Slack webhook configured.  
    - Edge Cases: Slack webhook failures, message formatting errors.  
    - Version: 2.2

  - **Publish to WordPress**  
    - Type: WordPress Node  
    - Role: Creates a new WordPress post titled with the preferred month and categorized under Travel, Japan, and Flight Deals.  
    - Configuration: Post title dynamically includes user’s preferred month; categories preselected.  
    - Inputs: Format Analysis Results output.  
    - Outputs: WordPress post creation confirmation.  
    - Credentials: WordPress API credentials required.  
    - Edge Cases: Authentication errors, API failures, category mismatches.  
    - Version: 1

  - **Respond to User**  
    - Type: Respond to Webhook  
    - Role: Sends a text response back to the user confirming completion of the analysis along with weather conditions and AI findings.  
    - Configuration: Response body includes formatted current weather for Tokyo, Osaka, Sapporo and summarized AI analysis, plus notes on Slack and WordPress publication.  
    - Inputs: Format Analysis Results output.  
    - Outputs: HTTP response to initial form submitter.  
    - Edge Cases: Response timeouts, user disconnects.  
    - Version: 1.1

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                   | Input Node(s)             | Output Node(s)                   | Sticky Note                                                                                                         |
|-------------------------|----------------------------------|--------------------------------------------------|---------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| User Input Form          | Form Trigger                     | Captures user travel preferences via web form    | —                         | Extract Form Data                | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Extract Form Data        | Set                              | Extracts and normalizes form data                  | User Input Form            | Scrape Flight Prices             | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Scrape Flight Prices     | HTTP Request                    | Scrapes Google Flights live prices                 | Extract Form Data          | Extract Flight Data              | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Extract Flight Data      | HTML Extract                    | Parses flight price data from scraped HTML         | Scrape Flight Prices       | Prepare AI Input                | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Prepare AI Input         | Set                              | Prepares flight and user preference data for AI   | Extract Flight Data        | AI Flight Analyzer, Fetch Weather Data | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Fetch Weather Data       | HTTP Request                    | Retrieves weather data for Japanese cities         | Prepare AI Input           | Parse Weather Data              | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Parse Weather Data       | Set                              | Structures weather data into city-specific formats | Fetch Weather Data         | Enrich AI Input with Weather    | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Enrich AI Input with Weather | Set                          | Combines flight, user, and weather data for AI    | Parse Weather Data         | AI Flight Analyzer              | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| AI Flight Analyzer       | LangChain Agent                  | Performs AI analysis on combined flight/weather data | Prepare AI Input, Enrich AI Input with Weather | Format Analysis Results        | Form Input → Extract Data → Scrape Flight Prices → Extract Pricing → Fetch Weather → Parse Weather → Prepare AI Input → AI Analysis → Format Results → Publish → Slack → Response |
| Structured Output Parser | LangChain Output Parser Structured | Parses AI output into structured JSON             | AI Flight Analyzer (parser input) | AI Flight Analyzer             |                                                                                                                     |
| OpenAI Chat Model        | LangChain OpenAI Chat Model     | Provides GPT model for AI analysis                  | AI Flight Analyzer (languageModel) | AI Flight Analyzer            |                                                                                                                     |
| Format Analysis Results  | Set                              | Consolidates AI output and metadata for output     | AI Flight Analyzer         | Send to Slack, Publish to WordPress, Respond to User |                                                                                                                     |
| Send to Slack            | Slack Node                      | Notifies Slack channel of analysis results          | Format Analysis Results    | —                               |                                                                                                                     |
| Publish to WordPress     | WordPress Node                  | Publishes analysis as a blog post                    | Format Analysis Results    | —                               |                                                                                                                     |
| Respond to User          | Respond to Webhook              | Sends confirmation and results back to user         | Format Analysis Results    | —                               |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create User Input Form Node:**  
   - Type: Form Trigger  
   - Set Path to `japan-flight-analyzer`  
   - Title: "Japan Flight Price Analyzer"  
   - Add fields:  
     - Departure City (text, required)  
     - Preferred Month (text, required)  
     - Maximum Budget (number, required)  
     - Flexible Dates (dropdown with options "Yes" and "No", required)  
   - Set response mode to "responseNode".

2. **Add Set Node "Extract Form Data":**  
   - Extract and assign variables:  
     - `departureCity` from form JSON  
     - `preferredMonth` from form JSON  
     - `maxBudget` as number  
     - `flexibleDates`  
   - Connect output of User Input Form to this node.

3. **Add HTTP Request Node "Scrape Flight Prices":**  
   - Method: GET  
   - URL: `https://www.google.com/travel/flights`  
   - Add query parameter `q` with value:  
     - Expression: `flights to Tokyo Japan from {{ $json.departureCity }}`  
   - Connect output of Extract Form Data to this node.

4. **Add HTML Extract Node "Extract Flight Data":**  
   - Operation: Extract HTML Content  
   - Configure selectors to parse flight price and relevant data (requires manual setup based on Google Flights HTML structure)  
   - Connect output of Scrape Flight Prices to this node.

5. **Add Set Node "Prepare AI Input":**  
   - Assign:  
     - `flightData` = raw content from Extract Flight Data’s response body  
     - `userPreferences` = formatted string combining departure city, month, budget, and flexibility using expressions  
   - Connect output of Extract Flight Data to this node.

6. **Add HTTP Request Node "Fetch Weather Data":**  
   - Method: GET  
   - URL: `https://api.open-meteo.com/v1/forecast`  
   - Query parameters:  
     - `latitude` = `35.6762,34.6937,43.0642`  
     - `longitude` = `139.6503,135.5023,141.3469`  
     - `current` = `temperature_2m,weather_code,wind_speed_10m`  
     - `daily` = `temperature_2m_max,temperature_2m_min,precipitation_sum,weather_code`  
     - `forecast_days` = `7`  
     - `timezone` = `Asia/Tokyo`  
   - Connect output of Prepare AI Input to this node.

7. **Add Set Node "Parse Weather Data":**  
   - Assign:  
     - `tokyoWeather` = first element of `current` array  
     - `osakaWeather` = second element of `current` array  
     - `sapporoWeather` = third element of `current` array  
     - `weatherForecast` = `daily` object  
   - Connect output of Fetch Weather Data to this node.

8. **Add Set Node "Enrich AI Input with Weather":**  
   - Assign `combinedData` as a multiline string concatenating:  
     - Flight Data (from Prepare AI Input)  
     - User Preferences (from Prepare AI Input)  
     - Current weather details for Tokyo, Osaka, Sapporo (temperature and wind speed from Parse Weather Data)  
     - 7-day forecast JSON string  
   - Connect output of Parse Weather Data to this node.

9. **Add LangChain OpenAI Chat Model Node "OpenAI Chat Model":**  
   - Model: `gpt-realtime`  
   - Credentials: Configure OpenAI API key  
   - No additional options needed.

10. **Add LangChain Agent Node "AI Flight Analyzer":**  
    - Text prompt: Include instructions to analyze flight data, user preferences, and weather info (use the combinedData variable).  
    - System message: Define AI as expert travel analyst.  
    - Enable structured output parser.  
    - Connect outputs:  
      - Language model input: connect OpenAI Chat Model output.  
      - Main input: connect Enrich AI Input with Weather output.

11. **Add LangChain Structured Output Parser Node "Structured Output Parser":**  
    - Connect AI Flight Analyzer’s output parser port to this node.

12. **Add Set Node "Format Analysis Results":**  
    - Assign:  
      - `aiAnalysis` = parsed AI output  
      - `userEmail` = form submission timestamp or user email if available  
      - `weatherData` = parsed weather info from Parse Weather Data  
    - Connect output of AI Flight Analyzer to this node.

13. **Add Slack Node "Send to Slack":**  
    - Configure Slack webhook credentials  
    - Compose message including:  
      - Search criteria (departure city, month, budget)  
      - Weather summaries for Tokyo, Osaka, Sapporo  
      - AI analysis summary  
    - Connect output of Format Analysis Results to this node.

14. **Add WordPress Node "Publish to WordPress":**  
    - Configure WordPress credentials for API access  
    - Set post title with preferred month included  
    - Assign categories: Travel, Japan, Flight Deals  
    - Connect output of Format Analysis Results to this node.

15. **Add Respond to Webhook Node "Respond to User":**  
    - Configure HTTP response with summary text including weather and AI analysis  
    - Connect output of Format Analysis Results to this node.

16. **Wiring:**  
    - Connect nodes in order as per above steps.  
    - Ensure parallel triggers are managed so AI Flight Analyzer waits for enriched data.  
    - Verify all credentials and API access tokens are properly configured.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Automates flight deal discovery and intelligent analysis for travel bloggers and deal hunters. Scrapes live pricing, enriches with weather data, applies AI evaluation, and auto-publishes to WordPress—eliminating manual research and accelerating content delivery. | Workflow Introduction Sticky Note |
| Prerequisites include: n8n instance, Google Flights access, weather API key, OpenAI-compatible AI service, WordPress site with API access, Slack workspace. | Prerequisites Sticky Note |
| Use cases include travel blog automation, flight deal newsletters, price comparison services, seasonal travel planning, destination weather analysis, and automated social media content. | Prerequisites Sticky Note |
| Customization options: Modify AI analysis criteria, adjust weather impact weighting, customize WordPress post templates, add email distribution, integrate additional data sources, expand to hotel/rental deals. | Prerequisites Sticky Note |
| Benefits: Eliminates manual price checking, combines multiple data sources automatically, delivers AI-enhanced insights, accelerates publishing workflow, scales across unlimited routes, provides weather-aware recommendations. | Prerequisites Sticky Note |
| Setup instructions: Configure form fields, connect Google Flights scraping endpoint, setup weather API and OpenAI credentials, set WordPress credentials and Slack webhook, define AI analysis prompts and output parsing rules. | Setup Instructions Sticky Note |
| Workflow steps summarized: Data collection via form and APIs, AI processing with enriched data, publishing results to WordPress, Slack notification, and user response. | Workflow Steps Sticky Note |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---