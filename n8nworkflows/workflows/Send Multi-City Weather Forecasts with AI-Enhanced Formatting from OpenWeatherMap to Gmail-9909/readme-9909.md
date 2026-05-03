Send Multi-City Weather Forecasts with AI-Enhanced Formatting from OpenWeatherMap to Gmail

https://n8nworkflows.xyz/workflows/send-multi-city-weather-forecasts-with-ai-enhanced-formatting-from-openweathermap-to-gmail-9909


# Send Multi-City Weather Forecasts with AI-Enhanced Formatting from OpenWeatherMap to Gmail

### 1. Workflow Overview

This workflow automates the generation and delivery of multi-city 5-day weather forecasts via email. It targets users requiring consolidated weather briefings for field operations, travel planning, outdoor events, logistics, or recreational activities. The workflow collects up to three city names through a web form, fetches and processes their weather forecasts from OpenWeatherMap (OWM), leverages an AI language model to compose a professionally formatted HTML email with condition-based color coding and safety tips, validates the AI output, and finally sends the email via Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Input & Normalization:** Collect city names from user input and normalize them for API requests.
- **1.2 Geocoding:** Convert city names to geographic coordinates using OWM Geocoding API.
- **1.3 Pairing & Location Preparation:** Match normalized cities with their coordinates, prepare data for forecast retrieval.
- **1.4 Forecast Retrieval & Summarization:** Fetch 5-day weather data per city from OWM and summarize daily weather points into concise data.
- **1.5 AI Processing & Formatting:** Pass summarized data to an AI model (GPT-4) to generate a well-structured, styled HTML email briefing and subject.
- **1.6 Result Validation:** Validate and safely extract the AI-generated JSON output.
- **1.7 Delivery:** Send the validated email briefing to the recipient via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Normalization

**Overview:**  
This block receives up to three city names from a form submission, filters and trims the input, and prepares individual items for downstream processing.

**Nodes Involved:**  
- TRIGGER - Input City Names  
- JS - Normalize City Inputs

**Node Details:**  

- **TRIGGER - Input City Names**  
  - Type: Form Trigger  
  - Role: Entry point; collects up to 3 city names from a web form with required validations.  
  - Configuration: Three text input fields labeled for city names, form title and description provided.  
  - Input: User-submitted form data.  
  - Output: JSON object with city names and metadata fields.  
  - Failure cases: Empty or invalid city name inputs; these are prevented by required field settings.  
  - No special version or sub-workflow requirements.

- **JS - Normalize City Inputs**  
  - Type: JavaScript Code  
  - Role: Extracts only city name strings from the form data, trims whitespace, excludes metadata keys, limits to three cities, and outputs one item per city.  
  - Configuration: Custom JS code that iterates over form fields, excludes `submittedAt` and `formMode`, collects non-empty trimmed strings, and slices to max 3.  
  - Expressions: Uses `$json` for input JSON access.  
  - Input: Raw form data JSON.  
  - Output: Array of items, each with a single city string property.  
  - Edge cases: If no valid city names, downstream nodes may fail due to empty inputs.  
  - No sub-workflow.

---

#### 1.2 Geocoding

**Overview:**  
This block resolves each city name into geographic coordinates (latitude and longitude) using the OpenWeatherMap Geocoding API.

**Nodes Involved:**  
- HTTP - OWM Geocoding

**Node Details:**  

- **HTTP - OWM Geocoding**  
  - Type: HTTP Request  
  - Role: Calls OWM Geocoding API with city name query, limits results to 1, and includes the API key.  
  - Configuration: GET request to `https://api.openweathermap.org/geo/1.0/direct` with query parameters: `q` (encoded city name), `limit=1`, and `appid` (API key).  
  - Expressions: Uses `encodeURIComponent($json.city)` to encode city name safely.  
  - Input: Items each containing a city name.  
  - Output: API JSON response with geolocation data.  
  - Failure modes: API key invalid, rate limits, empty or ambiguous city results, network errors.  
  - Requires valid OpenWeatherMap API credentials.

---

#### 1.3 Pairing & Location Preparation

**Overview:**  
This block aligns each normalized city with its corresponding geocoded coordinates and prepares structured data including city name, country, latitude, and longitude for forecast retrieval.

**Nodes Involved:**  
- MERGE - Pair Cities  
- JS - Pick Lat/Lon  
- SET - Carry Asked City to Forecast

**Node Details:**  

- **MERGE - Pair Cities**  
  - Type: Merge (Combine)  
  - Role: Combines two input streams by position, pairing normalized city names with geocoding results.  
  - Configuration: Combine mode by position, excludes unpaired items.  
  - Input: Left - normalized city items; Right - geocoding API responses.  
  - Output: Paired items with both city names and geocode data.  
  - Failure: Drop items if counts mismatch; could lose data if geocoding fails.  
  - No version or sub-workflow requirements.

- **JS - Pick Lat/Lon**  
  - Type: JavaScript Code  
  - Role: Extracts lat/lon and other location info from the geocoding API response, validates results, and packages into a consistent structure.  
  - Configuration: Custom JS that checks for valid lat/lon, throws error if missing, returns object with original asked city, API city name, country, lat, lon.  
  - Input: Merged city and geocode data.  
  - Output: Object with `asked_city`, `city_from_api`, `country`, `lat`, `lon`.  
  - Edge cases: Throws error if geocoding result is invalid or empty.  
  - No sub-workflow.

- **SET - Carry Asked City to Forecast**  
  - Type: Set  
  - Role: Passes along the `asked_city` property to be used in the next forecast node.  
  - Configuration: Simple assignment of `asked_city` from previous node.  
  - Input/Output: Single item flow-through.  
  - No failure modes.  

---

#### 1.4 Forecast Retrieval & Summarization

**Overview:**  
This block fetches the 5-day weather forecast for each city by name from OpenWeatherMap and processes the raw data into daily summaries including temperature ranges, precipitation, wind, and dominant weather conditions.

**Nodes Involved:**  
- OpenWeatherMap (OWM) - 5 Day Forecast (by City)  
- JS - Build Daily Summaries (5 days)  
- JS - Bundle Cities for LLM

**Node Details:**  

- **OpenWeatherMap (OWM) - 5 Day Forecast (by City)**  
  - Type: OpenWeatherMap node  
  - Role: Retrieves 5-day weather forecast by city name using OWM API.  
  - Configuration: Operation set to 5DayForecast, city name set dynamically from `asked_city`.  
  - Credentials: Requires OpenWeatherMap API key.  
  - Input: Item with `asked_city`.  
  - Output: Raw API forecast JSON.  
  - Failure: API key issues, invalid city, network errors, empty/no data returned.  

- **JS - Build Daily Summaries (5 days)**  
  - Type: JavaScript Code  
  - Role: Converts raw forecast data into aggregated daily summaries including min/max temperature, rain, wind gusts, humidity, precipitation probability, and dominant weather condition. Limits output to next 5 days.  
  - Configuration: Custom JS performing date grouping, calculations, rounding, and formatting of summary objects per day.  
  - Input: Raw OWM forecast JSON.  
  - Output: Object with `city`, `country`, `timezone_offset_seconds`, and array `summaries` of daily data.  
  - Edge cases: Missing or incomplete data fields; fallback defaults used.  
  - No sub-workflow.

- **JS - Bundle Cities for LLM**  
  - Type: JavaScript Code  
  - Role: Combines all city summary items into a single JSON payload to send to the AI model.  
  - Configuration: Maps all items into an array of city objects with relevant data.  
  - Input: Multiple city summary items.  
  - Output: Single item containing `cities` array.  
  - No failure modes expected.

---

#### 1.5 AI Processing & Formatting

**Overview:**  
This block sends the bundled city summaries to an AI language model (OpenAI GPT) to generate a professional weather briefing email subject and styled HTML content with color-coded condition rows and detailed safety tips.

**Nodes Involved:**  
- LLM - AI Weather Briefing Composer

**Node Details:**  

- **LLM - AI Weather Briefing Composer**  
  - Type: OpenAI (LangChain node)  
  - Role: Uses GPT-4o-mini model to generate a JSON response containing email subject and HTML content for all cities.  
  - Configuration: System message prompts expert meteorologist role with detailed instructions on structure, styling, color codes, precautions, and content guidelines. Uses precise JSON output format.  
  - Input: JSON payload with `cities` array containing weather summaries.  
  - Output: JSON string with `subject` and `html` keys.  
  - Failure modes: API key issues, rate limits, malformed input, AI output not valid JSON (mitigated by next validation node).  
  - Requires valid OpenAI API credentials.

---

#### 1.6 Result Validation

**Overview:**  
This block validates and extracts the AI-generated JSON output to ensure it contains the required fields before sending.

**Nodes Involved:**  
- JS - Validate JSON from LLM

**Node Details:**  

- **JS - Validate JSON from LLM**  
  - Type: JavaScript Code  
  - Role: Parses AI output to extract `subject` and `html` fields, verifies the JSON structure; returns success or failure.  
  - Configuration: Custom JS function handling multiple possible nested message formats from LLM output, attempts JSON parsing, returns `{ok: true}` with extracted data or `{ok: false}` with reason and raw text.  
  - Input: AI LLM output item.  
  - Output: Validated JSON object with `subject`, `html`, or failure info.  
  - Edge cases: Parsing failures, malformed AI responses, missing keys.  
  - No sub-workflow.

---

#### 1.7 Delivery

**Overview:**  
This block sends the validated weather briefing email via Gmail to the intended recipient.

**Nodes Involved:**  
- GMAIL - Send Weather Briefing

**Node Details:**  

- **GMAIL - Send Weather Briefing**  
  - Type: Gmail node  
  - Role: Sends an email with subject and HTML content produced by the AI.  
  - Configuration: Uses dynamic expressions to set `sendTo` (static placeholder), `subject`, and `message` (HTML) from validated LLM JSON output.  
  - Credentials: Requires Gmail OAuth2 authorization.  
  - Input: Validated email content with `subject` and `html`.  
  - Output: Email sent status.  
  - Failure modes: Authentication errors, quota exceeded, invalid recipient address, connectivity issues.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                               | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                           |
|----------------------------------|-------------------------|-----------------------------------------------|-----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| TRIGGER - Input City Names        | Form Trigger            | Collect user input of 3 city names             | None                              | JS - Normalize City Inputs         | ## Input & Normalization: Collect 3 city names from the form and normalize (trim, title-case, dedupe). |
| JS - Normalize City Inputs        | JavaScript Code         | Normalize and prepare city names               | TRIGGER - Input City Names         | HTTP - OWM Geocoding, MERGE - Pair Cities (secondary) |                                                                                                     |
| HTTP - OWM Geocoding              | HTTP Request            | Resolve city names to lat/lon via OWM API      | JS - Normalize City Inputs         | MERGE - Pair Cities                 | ## Geocoding (OWM): Resolve each city to lat/lon using OpenWeather Geocoding API.                   |
| MERGE - Pair Cities               | Merge                   | Pair normalized cities with geocoding results  | HTTP - OWM Geocoding, JS - Normalize City Inputs (2nd input) | JS - Pick Lat/Lon                  | ## Pairing Cities ↔ Coordinates: Align normalized city with its geocode (by position) and select asked_city, country, lat, lon. |
| JS - Pick Lat/Lon                 | JavaScript Code         | Extract and validate lat/lon and location info | MERGE - Pair Cities                | SET - Carry Asked City to Forecast |                                                                                                     |
| SET - Carry Asked City to Forecast| Set                     | Pass city name forward for forecast retrieval  | JS - Pick Lat/Lon                  | OpenWeatherMap (OWM) - 5 Day Forecast (by City) |                                                                                                     |
| OpenWeatherMap (OWM) - 5 Day Forecast (by City) | OpenWeatherMap node     | Fetch 5-day forecast data per city             | SET - Carry Asked City to Forecast | JS - Build Daily Summaries (5 days) | ## Forecast Retrieval: Pull 5-day forecast per city by coordinates.                                 |
| JS - Build Daily Summaries (5 days) | JavaScript Code       | Aggregate raw forecast into daily summaries    | OpenWeatherMap (OWM) - 5 Day Forecast (by City) | JS - Bundle Cities for LLM          | ## Summarization & Packaging: Convert raw OWM output into per-day tables & guidance; bundle all cities into one payload. |
| JS - Bundle Cities for LLM        | JavaScript Code         | Bundle all city summaries into one payload     | JS - Build Daily Summaries (5 days) | LLM - AI Weather Briefing Composer |                                                                                                     |
| LLM - AI Weather Briefing Composer| OpenAI (LangChain)      | Generate formatted email subject and HTML      | JS - Bundle Cities for LLM         | JS - Validate JSON from LLM         | ## LLM AI Weather Briefing: Ask LLM to produce subject + HTML for an email; validate/extract the result. |
| JS - Validate JSON from LLM       | JavaScript Code         | Validate and extract AI JSON output             | LLM - AI Weather Briefing Composer | GMAIL - Send Weather Briefing       |                                                                                                     |
| GMAIL - Send Weather Briefing     | Gmail                   | Send the final weather briefing email           | JS - Validate JSON from LLM        | None                              | ## Delivery: Send the formatted briefing to the recipient(s).                                      |
| Sticky Note                      | Sticky Note             | Documentation and overview                       | None                              | None                              | # Try Out overview, use cases, setup, and help links.                                               |
| Sticky Note1                     | Sticky Note             | Geocoding explanation                            | None                              | None                              | ## Geocoding (OWM): Resolve each city to lat/lon using OpenWeather Geocoding API.                   |
| Sticky Note2                     | Sticky Note             | Explanation of city-coordinate pairing           | None                              | None                              | ## Pairing Cities ↔ Coordinates: Align normalized city with its geocode (by position) and select asked_city, country, lat, lon. |
| Sticky Note3                     | Sticky Note             | Forecast retrieval explanation                    | None                              | None                              | ## Forecast Retrieval: Pull 5-day forecast per city by coordinates.                               |
| Sticky Note4                     | Sticky Note             | LLM AI briefing explanation                       | None                              | None                              | ## LLM AI Weather Briefing: Ask LLM to produce subject + HTML for an email; validate/extract the result. |
| Sticky Note5                     | Sticky Note             | Summarization and packaging explanation           | None                              | None                              | ## Summarization & Packaging: Convert raw OWM output into per-day tables & guidance; bundle all cities into one payload. |
| Sticky Note6                     | Sticky Note             | Delivery explanation                              | None                              | None                              | ## Delivery: Send the formatted briefing to the recipient(s).                                      |
| Sticky Note7                     | Sticky Note             | Input & normalization explanation                 | None                              | None                              | ## Input & Normalization: Collect 3 city names from the form and normalize (trim, title-case, dedupe). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `TRIGGER - Input City Names`  
   - Type: Form Trigger  
   - Configure fields: Three text fields for city names, all required.  
   - Set form title and description as in the original.  

2. **Add JavaScript Code Node**  
   - Name: `JS - Normalize City Inputs`  
   - Type: Code  
   - Paste the JS code to filter and trim city names, exclude metadata, limit to 3, output one item per city.  
   - Connect input from `TRIGGER - Input City Names`.  

3. **Add HTTP Request Node for Geocoding**  
   - Name: `HTTP - OWM Geocoding`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.openweathermap.org/geo/1.0/direct`  
   - Query Parameters:  
     - `q`: expression `={{ encodeURIComponent($json.city) }}`  
     - `limit`: `1`  
     - `appid`: Your OpenWeatherMap API key (credential or direct input)  
   - Connect input from `JS - Normalize City Inputs`.  

4. **Add Merge Node**  
   - Name: `MERGE - Pair Cities`  
   - Type: Merge (Combine)  
   - Mode: Combine by position  
   - Include unpaired: No  
   - Connect primary input from `HTTP - OWM Geocoding` output  
   - Connect secondary input from `JS - Normalize City Inputs` output (second input)  

5. **Add JavaScript Code Node**  
   - Name: `JS - Pick Lat/Lon`  
   - Type: Code  
   - Paste JS code that extracts lat/lon, validates, and returns structured location object.  
   - Connect input from `MERGE - Pair Cities`.  

6. **Add Set Node**  
   - Name: `SET - Carry Asked City to Forecast`  
   - Type: Set  
   - Assign `asked_city` from input JSON property `asked_city`.  
   - Connect input from `JS - Pick Lat/Lon`.  

7. **Add OpenWeatherMap Node**  
   - Name: `OpenWeatherMap (OWM) - 5 Day Forecast (by City)`  
   - Type: OpenWeatherMap  
   - Operation: `5DayForecast`  
   - City Name: expression `={{ $json.asked_city }}`  
   - Credentials: Your OpenWeatherMap API key.  
   - Connect input from `SET - Carry Asked City to Forecast`.  

8. **Add JavaScript Code Node**  
   - Name: `JS - Build Daily Summaries (5 days)`  
   - Type: Code  
   - Paste JS code to group forecast points by day and generate summaries including min/max temps, rain, wind, humidity, dominant condition, and formatting.  
   - Connect input from `OpenWeatherMap (OWM) - 5 Day Forecast (by City)`.  

9. **Add JavaScript Code Node**  
   - Name: `JS - Bundle Cities for LLM`  
   - Type: Code  
   - Paste JS code that bundles all city summary items into one JSON array `cities`.  
   - Connect inputs from all `JS - Build Daily Summaries (5 days)` nodes (if multiple cities, ensure all items feed here).  

10. **Add OpenAI Node (LangChain)**  
    - Name: `LLM - AI Weather Briefing Composer`  
    - Type: OpenAI (LangChain)  
    - Model: `gpt-4o-mini`  
    - Messages: Paste the full system and user prompt from the original, including detailed instructions on JSON structure, styling, and content.  
    - JSON Output: Enabled  
    - Credentials: OpenAI API key.  
    - Connect input from `JS - Bundle Cities for LLM`.  

11. **Add JavaScript Code Node**  
    - Name: `JS - Validate JSON from LLM`  
    - Type: Code  
    - Paste JS code that extracts and parses JSON from LLM output, checks for subject and html keys, returns validated object or error.  
    - Connect input from `LLM - AI Weather Briefing Composer`.  

12. **Add Gmail Node**  
    - Name: `GMAIL - Send Weather Briefing`  
    - Type: Gmail  
    - Credentials: Gmail OAuth2 with appropriate scopes to send email.  
    - Send To: Fixed or dynamic recipient email address.  
    - Subject: expression `={{ $json.subject }}` (from validated JSON)  
    - Message (HTML): expression `={{ $json.html }}`  
    - Connect input from `JS - Validate JSON from LLM`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                      | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates weather forecast delivery by collecting city names, fetching 5-day forecasts from OpenWeatherMap, and generating professionally formatted HTML emails using GPT-4.          | Workflow Overview Sticky Note                                                                    |
| The AI-generated emails use condition-based color coding with inline CSS, and include safety warnings for heavy rain, heat, wind, and thunderstorms.                                              | LLM prompt instructions in the AI Weather Briefing Composer node                                |
| Use OpenWeatherMap API key, OpenAI API key, and Gmail OAuth2 credentials with suitable permissions for email sending.                                                                             | Setup Requirements                                                                                 |
| For support or community discussion, join the Discord or visit the Forum. A README file is available at https://tinyurl.com/MulticityWeatherForecast                                               | Help and Documentation Link                                                                       |
| Geocoding is performed using OpenWeatherMap Geocoding API with query parameters: city name, limit=1, and API key.                                                                                 | Sticky Note1                                                                                      |
| City names are paired with geocode results by position to maintain correct alignment.                                                                                                              | Sticky Note2                                                                                      |
| Weather data is retrieved per city for 5 days, then processed into daily aggregates including temperature min/max, precipitation, wind, humidity, and dominant weather condition.                  | Sticky Note3                                                                                      |
| The AI prompt carefully instructs the model to produce strict JSON with subject and styled HTML content, including color-coded rows and detailed tips and precautions columns.                     | Sticky Note4                                                                                      |
| Raw API weather data is summarized and bundled before sending to the AI for email composition.                                                                                                    | Sticky Note5                                                                                      |
| The final email is sent to specified recipients using Gmail node with OAuth2 authentication.                                                                                                      | Sticky Note6                                                                                      |
| Input form collects and normalizes city names, excluding metadata fields and trimming whitespace, limiting to 3 cities.                                                                           | Sticky Note7                                                                                      |

---

**Disclaimer:**  
The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.