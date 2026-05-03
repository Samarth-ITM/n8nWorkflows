Generate Market Research Reports from Google Maps Reviews with Gemini AI

https://n8nworkflows.xyz/workflows/generate-market-research-reports-from-google-maps-reviews-with-gemini-ai-6928


# Generate Market Research Reports from Google Maps Reviews with Gemini AI

### 1. Workflow Overview

This n8n workflow automates the generation of detailed market research reports based on Google Maps business reviews, leveraging AI analysis via Google Gemini (PaLM). It targets business analysts, market researchers, and decision-makers seeking actionable insights about specific business categories in defined locations. The workflow is structured into logical blocks that handle input configuration, dynamic data retrieval from Google Maps, data extraction and enhancement, iterative review fetching, data aggregation and preparation, AI-powered market analysis, email report generation, and final logging.

Logical Blocks:

- **1.1 Input Configuration:** Captures user-defined search parameters such as query, location, language, and analysis focus.
- **1.2 Google Maps Data Retrieval:** Queries SerpAPI to dynamically fetch places matching the search criteria.
- **1.3 Data Extraction & Filtering:** Processes raw search results to extract enriched, validated place data.
- **1.4 Iterative Review Data Collection:** Loops over filtered places to fetch detailed customer reviews.
- **1.5 Data Combination & Aggregation:** Combines place metadata with review content and aggregates all data.
- **1.6 Analysis Data Preparation:** Structures the aggregated data into a detailed prompt for AI analysis.
- **1.7 AI Market Analysis:** Uses Google Gemini to generate an in-depth, structured market research report in Vietnamese.
- **1.8 Email Report Preparation & Delivery:** Formats the AI report into HTML and plain text, then sends it via Gmail.
- **1.9 Final Logging:** Logs workflow execution details and results for auditing and monitoring.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Configuration

- **Overview:** Defines and sets the core search parameters driving the entire workflow.
- **Nodes Involved:**  
  - User Input Configuration
- **Node Details:**

  - **User Input Configuration**  
    - Type: Set node  
    - Role: Initializes user parameters such as search query, location coordinates with zoom, language code, analysis focus (business category), and city name.  
    - Configuration: Fields set statically with example values (e.g., "nhà hàng quận 1 TP.HCM" for search query, coordinates for Ho Chi Minh City).  
    - Expressions: None; values are set directly.  
    - Input/Output: Input from manual trigger → output to Dynamic Search Places node.  
    - Edge Cases: Incorrect or missing parameters could lead to no data returned or API errors.  
    - Version: 3.2  

#### 1.2 Google Maps Data Retrieval

- **Overview:** Sends a dynamic HTTP request to SerpAPI to search Google Maps places matching user parameters.
- **Nodes Involved:**  
  - Dynamic Search Places
- **Node Details:**

  - **Dynamic Search Places**  
    - Type: HTTP Request  
    - Role: Queries SerpAPI's Google Maps search endpoint.  
    - Configuration:  
      - URL: `https://serpapi.com/search.json`  
      - Query parameters dynamically constructed from User Input Configuration node:  
        - engine = "google_maps"  
        - q = search_query  
        - ll = search_location (latitude, longitude, zoom)  
        - hl = language_code  
        - api_key = SerpAPI key (hardcoded in workflow)  
      - Sends query as JSON, expects JSON response.  
    - Input/Output: Input from User Input Configuration → output to Enhanced Extract Data.  
    - Edge Cases: API key invalid or rate-limited; location or query malformed; network timeouts.  
    - Version: 4.2  

#### 1.3 Data Extraction & Filtering

- **Overview:** Processes the raw search results, extracts relevant place data with enhanced metadata, and filters out incomplete entries.
- **Nodes Involved:**  
  - Enhanced Extract Data
- **Node Details:**

  - **Enhanced Extract Data**  
    - Type: Code (JavaScript)  
    - Role: Parses local_results array from SerpAPI response; extracts multiple metadata fields including name, address, rating, reviews count, phone, website, coordinates, price level, operating hours, etc.  
    - Key Expressions: Accesses input JSON, maps each place to enriched object, filters places that have either review links or sufficient rating/review count.  
    - Input/Output: Input from Dynamic Search Places → output to Loop Over Places.  
    - Edge Cases: Missing or null fields handled by default values like 'Unknown' or null; empty results yield empty arrays.  
    - Version: 2  

#### 1.4 Iterative Review Data Collection

- **Overview:** Iterates over each filtered place to request detailed reviews from SerpAPI.
- **Nodes Involved:**  
  - Loop Over Places  
  - Get Reviews Content  
  - Enhanced Combine Data  
- **Node Details:**

  - **Loop Over Places**  
    - Type: SplitInBatches  
    - Role: Controls batch-wise processing of each place to avoid overload; splits input array into individual items.  
    - Input/Output: Input from Enhanced Extract Data → outputs to Collect All Data and Get Reviews Content.  
    - Edge Cases: Large numbers of places may cause long run times; batching helps control rate limits.  
    - Version: 3  

  - **Get Reviews Content**  
    - Type: HTTP Request  
    - Role: Fetches detailed reviews JSON for each place using the place's review link and SerpAPI key.  
    - Configuration:  
      - URL dynamically set from place's reviews_link field.  
      - Timeout set to 30 seconds.  
      - Sends API key in JSON query parameter.  
    - Input/Output: Input from Loop Over Places → output to Enhanced Combine Data.  
    - Edge Cases: Missing or invalid reviews_link; HTTP timeouts; API rate limits; unauthorized certificates allowed.  
    - Version: 4.2  

  - **Enhanced Combine Data**  
    - Type: Code (JavaScript)  
    - Role: Combines place metadata with fetched reviews content into a single enriched JSON object; calculates completeness score based on field presence.  
    - Key Expressions: Uses input JSON from Loop Over Places and Get Reviews Content; calculates completeness score as percentage of key fields filled.  
    - Input/Output: Input from Get Reviews Content and Loop Over Places → output to Loop Over Places (for batch continuation) and Collect All Data.  
    - Edge Cases: Reviews content may be missing or empty; handled by including an error object.  
    - Version: 2  

#### 1.5 Data Combination & Aggregation

- **Overview:** Aggregates all enriched place data into a single collection for final analysis.
- **Nodes Involved:**  
  - Collect All Data
- **Node Details:**

  - **Collect All Data**  
    - Type: Aggregate node  
    - Role: Gathers all batch-processed place data into a single array for downstream processing.  
    - Input/Output: Input from Loop Over Places → output to Prepare Analysis Data.  
    - Edge Cases: Empty inputs produce empty arrays; ensures all data is collected before advancing.  
    - Version: 1  

#### 1.6 Analysis Data Preparation

- **Overview:** Prepares a comprehensive textual prompt summarizing all collected data and statistics for AI consumption.
- **Nodes Involved:**  
  - Prepare Analysis Data
- **Node Details:**

  - **Prepare Analysis Data**  
    - Type: Code (JavaScript)  
    - Role: Iterates over all place data; compiles a detailed report including place details, sample reviews, summary statistics (total reviews, average rating, data completeness), and context info.  
    - Key Expressions: Dynamically builds a multiline string prompt in Vietnamese detailing the dataset and statistics for AI analysis.  
    - Input/Output: Input from Collect All Data → output to AI Market Analysis.  
    - Edge Cases: Handles missing data gracefully; computes averages carefully to avoid division by zero.  
    - Version: 2  

#### 1.7 AI Market Analysis

- **Overview:** Uses Google Gemini Chat Model to generate a detailed market research report in Vietnamese, based on prepared prompt.
- **Nodes Involved:**  
  - AI Market Analysis  
  - Google Gemini Chat Model
- **Node Details:**

  - **AI Market Analysis**  
    - Type: LangChain Agent  
    - Role: Defines detailed instructions and task for AI to analyze the market data and generate a structured report with sections like executive summary, customer experience, competitive landscape, and actionable recommendations.  
    - Configuration: Uses template text with placeholders filled from user input and aggregated data.  
    - Input/Output: Input from Prepare Analysis Data → output to Prepare Email Content.  
    - Edge Cases: AI response variability; prompt length limits; requires valid API credentials.  
    - Version: 2.1  

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat node (Google Gemini)  
    - Role: Executes the AI model call using Google PaLM API credentials.  
    - Credentials: Uses Google Palm API key configured in n8n credentials.  
    - Input/Output: Input from AI Market Analysis → output back to AI Market Analysis node.  
    - Edge Cases: API quota limits; authentication errors; latency.  
    - Version: 1  

#### 1.8 Email Report Preparation & Delivery

- **Overview:** Formats the AI-generated report into professional HTML and plain text emails, and sends the report via Gmail.
- **Nodes Involved:**  
  - Prepare Email Content  
  - Send Email Report
- **Node Details:**

  - **Prepare Email Content**  
    - Type: Code (JavaScript)  
    - Role: Formats AI report content into a styled HTML email with headings, highlights, and responsive design; also creates a plain text fallback version. Constructs dynamic subject line.  
    - Input/Output: Input from AI Market Analysis → output to Send Email Report.  
    - Edge Cases: HTML rendering issues in some email clients; large content may cause email size limits.  
    - Version: 2  

  - **Send Email Report**  
    - Type: Gmail node  
    - Role: Sends the formatted email report to a configured recipient address (hardcoded to truong11062002@gmail.com).  
    - Credentials: Uses Gmail OAuth2 credentials configured in n8n.  
    - Input/Output: Input from Prepare Email Content → output to Final Status & Logging.  
    - Edge Cases: Email delivery failures; credential expiration; quota limits.  
    - Version: 2  

#### 1.9 Final Logging

- **Overview:** Logs summary information about the workflow execution and returns a structured JSON summary.
- **Nodes Involved:**  
  - Final Status & Logging
- **Node Details:**

  - **Final Status & Logging**  
    - Type: Code (JavaScript)  
    - Role: Logs key runtime info such as total locations analyzed, reviews processed, average rating, and email delivery success. Returns a comprehensive JSON object summarizing workflow metadata, data results, AI usage, and recommendations for next steps.  
    - Input/Output: Input from Send Email Report → terminal output.  
    - Edge Cases: Logging failures do not impact workflow; ensures comprehensive audit trail.  
    - Version: 2  

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                    |
|---------------------------|------------------------------------|----------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Start Workflow            | Manual Trigger                     | Entry point to start workflow           | —                           | User Input Configuration     |                                                                                                               |
| User Input Configuration  | Set                               | Defines search parameters                | Start Workflow              | Dynamic Search Places        |                                                                                                               |
| Dynamic Search Places     | HTTP Request                      | Queries Google Maps via SerpAPI          | User Input Configuration    | Enhanced Extract Data        |                                                                                                               |
| Enhanced Extract Data     | Code                              | Extracts and filters place data          | Dynamic Search Places       | Loop Over Places             |                                                                                                               |
| Loop Over Places          | SplitInBatches                    | Iterates over places for review fetching | Enhanced Extract Data, Enhanced Combine Data | Collect All Data, Get Reviews Content |                                                                                                               |
| Get Reviews Content       | HTTP Request                     | Fetches detailed reviews for each place | Loop Over Places            | Enhanced Combine Data        |                                                                                                               |
| Enhanced Combine Data     | Code                              | Combines place info with reviews         | Get Reviews Content, Loop Over Places | Loop Over Places             |                                                                                                               |
| Collect All Data          | Aggregate                        | Aggregates all place data                 | Loop Over Places            | Prepare Analysis Data        |                                                                                                               |
| Prepare Analysis Data     | Code                              | Prepares detailed analysis prompt         | Collect All Data            | AI Market Analysis           |                                                                                                               |
| AI Market Analysis        | LangChain Agent                   | Defines AI prompt for market analysis    | Prepare Analysis Data       | Prepare Email Content        |                                                                                                               |
| Google Gemini Chat Model  | LangChain LM Chat (Google Gemini) | Executes AI model call                     | AI Market Analysis          | AI Market Analysis (ai_languageModel) |                                                                                                               |
| Prepare Email Content     | Code                              | Formats AI report into HTML & plain text | AI Market Analysis          | Send Email Report            |                                                                                                               |
| Send Email Report         | Gmail                            | Sends report email                        | Prepare Email Content       | Final Status & Logging       |                                                                                                               |
| Final Status & Logging    | Code                              | Logs execution summary and outputs JSON  | Send Email Report           | —                           |                                                                                                               |
| Sticky Note              | Sticky Note                      | Documentation and quick start guide      | —                           | —                           | # 🚀 Market Research Analytics System\n\n> **Transform Google Maps data into actionable business insights with AI-powered analysis**\n\n## 📋 Overview\n\nThis n8n workflow automatically collects business data from Google Maps, analyzes customer reviews using AI, and generates comprehensive market research reports delivered straight to your inbox.\n\n---\n\n## ⚡ Quick Start Guide\n\n### Step 1: Configure Your Search Parameters\n\nNavigate to the **\"Configuration Variables\"** node and update these fields:\n\n```json\n{\n  \"search_query\": \"restaurants downtown\",        // Your target business type + location\n  \"search_location\": \"@40.7589,-73.9851,12z\",   // Coordinates (lat,lng,zoom)\n  \"language_code\": \"en\",                         // Language: en, vi, es, fr, etc.\n  \"analysis_focus\": \"restaurant\",                // Business category for analysis\n  \"city_name\": \"New York City\"                   // Target city name\n}\n```\n\n**🔍 Need coordinates?** Use [LatLong.net](https://www.latlong.net/) to find your target location coordinates.\n\n---\n\n### Step 2: Setup API Keys\n\n#### 🔑 SerpAPI (Google Maps Data)\n1. **Get API Key:** Visit [SerpAPI Google Maps Reviews](https://serpapi.com/google-maps-reviews-api)\n2. **Sign up** for free account (100 searches/month included)\n3. **Copy your API key** from the dashboard\n4. **Update workflow:** Replace `YOUR_SERPAPI_KEY_HERE` in both HTTP nodes\n\n#### 🤖 Google Gemini AI (Analysis)\n1. **Get API Key:** Visit [Google AI Studio](https://ai.google.dev/gemini-api/docs/api-key)\n2. **Create new API key** (free tier available)\n3. **Setup in n8n:** Add credentials in \"Google Gemini Chat Model\" node\n\n---\n\n### Step 3: Configure Email Delivery\n\n1. **Open** the **\"Send Email Report\"** node\n2. **Update recipient:** Change `your-email@example.com` to your email\n3. **Gmail setup:** Connect your Gmail account in n8n credentials\n\n---\n\n### Step 4: Execute & Enjoy! 🎉 |


---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node:**  
   - Name: "Start Workflow"  
   - Purpose: Entry point for manual execution.

2. **Create set node for user input:**  
   - Name: "User Input Configuration"  
   - Type: Set  
   - Parameters:  
     - search_query: e.g., "nhà hàng quận 1 TP.HCM"  
     - search_location: e.g., "@10.8231,106.6297,12z" (lat,lng,zoom)  
     - language_code: e.g., "vi"  
     - analysis_focus: e.g., "restaurant"  
     - city_name: e.g., "TP. Hồ Chí Minh"  
   - Connect: "Start Workflow" → "User Input Configuration"

3. **Create HTTP Request node to query SerpAPI:**  
   - Name: "Dynamic Search Places"  
   - URL: `https://serpapi.com/search.json`  
   - Method: GET  
   - Query parameters (JSON):  
     - engine: "google_maps"  
     - q: Expression referencing `search_query` from "User Input Configuration"  
     - ll: Expression referencing `search_location`  
     - hl: Expression referencing `language_code`  
     - api_key: Your SerpAPI key (replace with your own key)  
   - Connect: "User Input Configuration" → "Dynamic Search Places"

4. **Create code node to extract and filter places:**  
   - Name: "Enhanced Extract Data"  
   - Code: Implement JavaScript to parse `local_results` from SerpAPI response, extract detailed fields (name, rating, reviews count, phone, website, operating hours, coordinates, etc.), enrich with search context data, and filter for places with reviews link or rating.  
   - Connect: "Dynamic Search Places" → "Enhanced Extract Data"

5. **Create SplitInBatches node to loop over places:**  
   - Name: "Loop Over Places"  
   - Batch size: Default (1)  
   - Connect: "Enhanced Extract Data" → "Loop Over Places"

6. **Create HTTP Request node to fetch reviews:**  
   - Name: "Get Reviews Content"  
   - URL: Dynamic expression from current item's `reviews_link` field  
   - Method: GET  
   - Query parameters: `{ "api_key": "YOUR_SERPAPI_KEY" }`  
   - Timeout: 30000 ms  
   - Allow unauthorized certs: true  
   - Connect: "Loop Over Places" (secondary output) → "Get Reviews Content"

7. **Create code node to combine place data and reviews:**  
   - Name: "Enhanced Combine Data"  
   - Code: Combine current place metadata with fetched reviews JSON. Calculate completeness score based on presence of key fields. Handle missing reviews data by adding error object.  
   - Connect: "Get Reviews Content" → "Enhanced Combine Data"  
   - Connect: "Enhanced Combine Data" → "Loop Over Places" (to continue iteration)

8. **Create Aggregate node to collect all combined data:**  
   - Name: "Collect All Data"  
   - Aggregate mode: Aggregate all item data into array  
   - Connect: "Loop Over Places" (main output) → "Collect All Data"

9. **Create code node to prepare AI analysis prompt:**  
   - Name: "Prepare Analysis Data"  
   - Code: Iterate over aggregated place data; build a formatted prompt in Vietnamese including dataset overview, detailed place information, sample reviews, and summary statistics (e.g., total reviews, average rating, data completeness).  
   - Connect: "Collect All Data" → "Prepare Analysis Data"

10. **Create LangChain Agent node for AI prompt definition:**  
    - Name: "AI Market Analysis"  
    - Parameters: Detailed Vietnamese prompt instructions for Google Gemini, specifying report sections, analysis focus, and actionable recommendations.  
    - Connect: "Prepare Analysis Data" → "AI Market Analysis"

11. **Create LangChain LM Chat node for Google Gemini:**  
    - Name: "Google Gemini Chat Model"  
    - Credentials: Configure Google Palm API key in n8n credentials and select here  
    - Connect: "AI Market Analysis" (ai_languageModel output) → "Google Gemini Chat Model"

12. **Create code node to format email content:**  
    - Name: "Prepare Email Content"  
    - Code: Extract AI report content; generate dynamic subject line; convert markdown-like formatting to styled HTML email; create plain text fallback; include analysis metadata.  
    - Connect: "AI Market Analysis" → "Prepare Email Content"

13. **Create Gmail node to send email:**  
    - Name: "Send Email Report"  
    - Parameters:  
      - To: your email address (e.g., "truong11062002@gmail.com")  
      - Subject: Expression from "Prepare Email Content" subject  
      - Message: Expression from "Prepare Email Content" htmlContent  
    - Credentials: Connect Gmail OAuth2 credentials  
    - Connect: "Prepare Email Content" → "Send Email Report"

14. **Create code node for final logging and summary:**  
    - Name: "Final Status & Logging"  
    - Code: Log workflow status, data statistics, email delivery status; return JSON summary with metadata and next steps.  
    - Connect: "Send Email Report" → "Final Status & Logging"

15. **Optional:** Add Sticky Note node with workflow overview and quick start instructions for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| 🚀 Market Research Analytics System: Transform Google Maps data into actionable business insights with AI-powered analysis. Quick start guide included for configuring search parameters, API keys for SerpAPI and Google Gemini, and email setup. Use LatLong.net to find coordinates. Replace placeholder API keys with your own. Reports are generated in Vietnamese with professional formatting and actionable recommendations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Embedded sticky note inside workflow; external links: [SerpAPI Google Maps Reviews](https://serpapi.com/google-maps-reviews-api), [Google AI Studio API Key](https://ai.google.dev/gemini-api/docs/api-key), [LatLong.net](https://www.latlong.net/) |
| SerpAPI API key is hardcoded in HTTP Request nodes; for production use, replace with environment variables or credential manager for security.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Workflow best practice                                               |
| Google Gemini (PaLM) API credentials must be configured in n8n’s credential management and linked to the "Google Gemini Chat Model" node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Credential setup required                                            |
| Gmail OAuth2 credentials are required for sending email reports. Make sure permissions for sending emails are granted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Credential setup required                                            |
| The workflow handles multi-language support (Vietnamese example provided), but can be adapted by changing user input variables and AI prompt language.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Localization capability                                              |
| Rate limiting and API quota management should be considered due to multiple API calls per place (search + reviews). Batching and timeouts are implemented to mitigate overload.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Scalability consideration                                           |
| The AI prompt instructs Google Gemini to produce structured sections including executive summary, market overview, customer experience, competitive landscape, insights, strategy recommendations, and action plans with KPIs, ensuring high-quality actionable output.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | AI prompt design best practice                                       |
| The email formatting includes responsive HTML styling with semantic headings and highlights, ensuring readability across devices and clients.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | UX/UI design best practice                                           |

---

**Disclaimer:** The provided text is solely derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.