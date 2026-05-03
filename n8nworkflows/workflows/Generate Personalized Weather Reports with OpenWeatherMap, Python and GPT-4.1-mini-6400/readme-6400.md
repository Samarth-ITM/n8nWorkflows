Generate Personalized Weather Reports with OpenWeatherMap, Python and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/generate-personalized-weather-reports-with-openweathermap--python-and-gpt-4-1-mini-6400


# Generate Personalized Weather Reports with OpenWeatherMap, Python and GPT-4.1-mini

### 1. Workflow Overview

This workflow generates personalized weather reports for a user-specified city by integrating data from OpenWeatherMap, processing it with Python, enriching the content with GPT-4.1-mini AI, and finally sending the report via Gmail. The main use case is to automate the creation and delivery of detailed, readable weather emails that include analysis, recommendations, and a weather-related joke.

Logical blocks:

- **1.1 Input Reception:** Receives city input via a form submission.
- **1.2 Weather Data Retrieval:** Fetches current weather data from OpenWeatherMap API.
- **1.3 Weather Data Processing:** Processes raw weather data using Python to extract, analyze, and enhance information.
- **1.4 Email Content Generation:** Converts processed data into well-formatted HTML and plain text email content with Python.
- **1.5 AI Enrichment:** Uses an AI agent (GPT-4.1-mini) to polish the email content, add readability improvements, and insert a weather-related joke.
- **1.6 Email Sending:** Sends the final email via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user input (city name) through a web form to trigger the workflow.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (instructional)

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point of the workflow; waits for user input via a form titled "Enter a City" with a single field "City" (placeholder example: "New York").  
    - Configuration: Outputs JSON containing the city name entered as `$json.City`.  
    - Input: External HTTP request (form submission).  
    - Output: Triggers next node with city name data.  
    - Edge cases: Missing or invalid city name input, slow form submission.  

  - **Sticky Note**  
    - Provides instruction: "Enter the name of a city in the form" to guide the user or developer.

---

#### 1.2 Weather Data Retrieval

- **Overview:**  
  Fetches current weather data from OpenWeatherMap API using the city input from the form.

- **Nodes Involved:**  
  - Fetch Weather Data  
  - Sticky Note (API key instruction)

- **Node Details:**  
  - **Fetch Weather Data**  
    - Type: HTTP Request  
    - Role: Calls OpenWeatherMap's current weather API endpoint (`https://api.openweathermap.org/data/2.5/weather`).  
    - Configuration:  
      - Query parameters include:  
        - `q`: city name dynamically set from `$json.City`.  
        - `appid`: your OpenWeatherMap API key (must be configured).  
        - `units`: metric for Celsius temperatures.  
      - Sends GET request with query parameters.  
    - Input: JSON with city from form submission.  
    - Output: Raw weather data JSON from API.  
    - Edge cases: API key missing/invalid, city not found, network timeout, API rate limits.  
    - Sticky Note reminder: Replace `<your_API_key>` with your actual OpenWeatherMap API key.

---

#### 1.3 Weather Data Processing

- **Overview:**  
  Processes raw weather JSON data via a Python script to extract essential weather attributes, compute derived metrics (e.g., Fahrenheit conversion, comfort levels), and generate a detailed weather report object.

- **Nodes Involved:**  
  - Process Weather Data  
  - Sticky Note (Python processing and API key note)

- **Node Details:**  
  - **Process Weather Data**  
    - Type: Code (Python)  
    - Role: Parses raw API data and performs:  
      - Extraction of temperature, humidity, pressure, wind speed, visibility, description, location info.  
      - Conversion of temperature units (Celsius to Fahrenheit).  
      - Wind speed conversions (m/s to km/h and mph).  
      - Comfort level assessment based on temperature and humidity.  
      - Clothing suggestions and activity recommendations based on weather conditions.  
      - Weather emoji determination for visual appeal.  
      - Calculation of a weather quality score (0-100) summarizing overall weather favorability.  
      - Outputs a structured JSON object containing all processed and analyzed information, including raw data for reference.  
    - Key expressions: Uses Python functions internally for comfort, clothing, activity, and emoji determination; returns a JSON object with nested data.  
    - Input: Raw weather JSON from HTTP request node.  
    - Output: Processed and analyzed weather data JSON.  
    - Edge cases: Missing or malformed API data fields, division by zero (not expected), unexpected API response changes.  
    - Requires Python environment support in n8n.  

---

#### 1.4 Email Content Generation

- **Overview:**  
  Converts the processed weather data into formatted HTML and plain text email content, ready for sending.

- **Nodes Involved:**  
  - Generate Email Content  
  - Sticky Note (OpenAI API key instruction)

- **Node Details:**  
  - **Generate Email Content**  
    - Type: Code (Python)  
    - Role:  
      - Constructs a comprehensive HTML email with structured styling, sections for current conditions, recommendations, weather quality score (visualized as a conic gradient circle), and footer.  
      - Generates a plain text alternative email version for compatibility.  
      - Embeds dynamic data such as temperatures, humidity, wind speed, timestamps, comfort levels, and suggestions.  
      - The email subject includes weather emoji, location, and temperature.  
    - Input: Processed weather report JSON from Python processing node.  
    - Output: JSON with email subject, `html_content`, `text_content`, and embedded weather data.  
    - Edge cases: Faulty HTML generation if data fields missing, potential encoding issues with emojis.  

---

#### 1.5 AI Enrichment

- **Overview:**  
  Uses GPT-4.1-mini AI model to improve the email message by enhancing readability, formatting, and adding a relevant weather joke.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Sticky Note (OpenAI API key reminder)

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Receives plain text weather summary and instructions to format it nicely in HTML, add paragraphs, bullet points, and insert a weather-related joke.  
    - Configuration:  
      - Prompt includes the raw text content from previous node.  
      - Output: Polished HTML email body with added humor and formatting.  
    - Input: Plain text weather summary from "Generate Email Content" node.  
    - Output: Enhanced HTML email content for sending.  
    - Edge cases: API call failures, unexpected AI output formatting, rate limits on OpenAI side.  
    - Requires OpenAI API key credentials configured.  

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Language model provider for AI Agent. Uses GPT-4.1-mini model.  
    - Input/Output: Connects directly to AI Agent node as language model backend.  
    - Edge cases: Authentication errors, model availability.  

---

#### 1.6 Email Sending

- **Overview:**  
  Sends the finalized, AI-enhanced weather report email to a specified recipient using Gmail.

- **Nodes Involved:**  
  - Send a message  
  - Sticky Note (Gmail credential instructions)

- **Node Details:**  
  - **Send a message**  
    - Type: Gmail node  
    - Role: Sends the email with subject and message body (HTML) generated by AI Agent.  
    - Configuration:  
      - Recipient email address set dynamically (parameter: `emailAddress`).  
      - Subject set from "Generate Email Content" subject field.  
      - Message body is the output from AI Agent node.  
      - Uses Gmail OAuth2 credentials configured in n8n.  
    - Input: Polished email content and subject.  
    - Output: Confirmation of email sent.  
    - Edge cases: Credential invalidation, Gmail sending limits, network errors.  

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                     | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                   |
|---------------------|-------------------------------|-----------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| Sticky Note         | Sticky Note                   | Instruction for city input        |                        | On form submission      | Enter the name of a city in the form                                                          |
| On form submission  | Form Trigger                  | Receive city input via form       |                        | Fetch Weather Data      |                                                                                               |
| Sticky Note1        | Sticky Note                   | API key setup instruction         |                        | Fetch Weather Data      | Add OpenWeather API key to get current weather information about the city, replace <your_API_key> with your actual API key. Python script will process and generate custom email relating to the weather. |
| Fetch Weather Data  | HTTP Request                  | Retrieve weather data from API    | On form submission      | Process Weather Data    |                                                                                               |
| Process Weather Data| Code (Python)                 | Process and analyze weather data  | Fetch Weather Data      | Generate Email Content  |                                                                                               |
| Sticky Note2        | Sticky Note                   | OpenAI API key instruction        |                        | AI Agent                | Add OpenAI API Key. AI Agent will add a custom joke about the weather for the city you entered as part of the email message. |
| Generate Email Content | Code (Python)               | Generate HTML and text email body | Process Weather Data    | AI Agent                |                                                                                               |
| OpenAI Chat Model   | LangChain LM Chat OpenAI      | Language model for AI Agent       |                        | AI Agent                |                                                                                               |
| AI Agent            | LangChain Agent               | Enhance email content with AI     | Generate Email Content  | Send a message          |                                                                                               |
| Sticky Note3        | Sticky Note                   | Gmail credential and email setup  |                        | Send a message          | Add your Gmail credentials and update header and body of message. Node will send the custom email to the recipient |
| Send a message      | Gmail                        | Send final email                  | AI Agent                |                        |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node:**
   - Type: Form Trigger  
   - Configure form title "Enter a City"  
   - Add one field: Label "City", placeholder "New York"  
   - This node will output JSON with the key `$json.City`.

2. **Create "Fetch Weather Data" node:**
   - Type: HTTP Request  
   - Set method to GET  
   - URL: `https://api.openweathermap.org/data/2.5/weather`  
   - Add query parameters:  
     - `q`: Expression `={{ $json.City }}` (city name from form)  
     - `appid`: Your OpenWeatherMap API key (string, replace `<your_API_key>`)  
     - `units`: `"metric"`  
   - Connect "On form submission" → "Fetch Weather Data"

3. **Create "Process Weather Data" node:**
   - Type: Code (Python)  
   - Paste provided Python script that extracts weather info, calculates comfort levels, clothing/activity suggestions, weather emoji, and weather quality score.  
   - Connect "Fetch Weather Data" → "Process Weather Data"  
   - Ensure your n8n instance supports Python code execution.

4. **Create "Generate Email Content" node:**
   - Type: Code (Python)  
   - Paste provided Python script that builds HTML and text email content from processed weather data, including styling and dynamic data insertion.  
   - Connect "Process Weather Data" → "Generate Email Content"

5. **Create "AI Agent" node:**
   - Type: LangChain Agent  
   - Set prompt to instruct the agent to format the plain text weather summary into a polished HTML email with paragraphs, bullet points, and a weather joke. Use expression to include the plain text from "Generate Email Content" node.  
   - Connect "Generate Email Content" → "AI Agent"

6. **Create "OpenAI Chat Model" node:**
   - Type: LangChain LM Chat OpenAI  
   - Select model: "gpt-4.1-mini"  
   - Connect "OpenAI Chat Model" → "AI Agent" (as language model backend)  
   - Configure OpenAI API key credentials in n8n.

7. **Create "Send a message" node:**
   - Type: Gmail  
   - Configure Gmail OAuth2 credentials in n8n.  
   - Set "Send To" email address (can be static or dynamic parameter `emailAddress`)  
   - Subject: Use expression from "Generate Email Content" node's subject field.  
   - Message: Use output from "AI Agent" node (enhanced HTML content).  
   - Connect "AI Agent" → "Send a message"

8. **Add Sticky Notes as needed:**
   - To remind about entering city in form.  
   - To insert and replace OpenWeatherMap API key in HTTP Request node.  
   - To configure OpenAI API key for AI Agent.  
   - To configure Gmail credentials and update email headers/body.

9. **Activate and test the workflow:**  
   - Trigger the form submission with a valid city name.  
   - Confirm the weather data fetch, processing, AI enhancement, and successful email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow is powered by Python scripting for data processing and n8n nodes for integration.      | Workflow design detail                                             |
| OpenWeatherMap API key is required and must be obtained from https://openweathermap.org/api          | API key registration                                               |
| OpenAI API key with access to GPT-4.1-mini model is necessary for AI Agent functionality             | https://platform.openai.com/account/api-keys                       |
| Gmail OAuth2 credentials must be set up in n8n for email sending                                     | Google Cloud Console for OAuth2 credentials                        |
| Weather emoji and comfort analysis add user-friendly and engaging aspects to the weather report      | Enhances readability and user engagement                           |
| The HTML email uses inline CSS and responsive design for compatibility with most email clients       | Ensures good email rendering                                       |
| The AI Agent also adds a relevant weather joke to increase email personalization and user delight   | Improves user experience                                           |

---

**Disclaimer:**  
The provided text and workflow derive exclusively from an automated n8n workflow. All data processed is legal and public. No illegal, offensive, or protected content is involved. The workflow adheres strictly to applicable content policies.