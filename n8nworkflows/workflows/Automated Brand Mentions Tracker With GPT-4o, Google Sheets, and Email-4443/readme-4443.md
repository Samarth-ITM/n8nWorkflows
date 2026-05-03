Automated Brand Mentions Tracker With GPT-4o, Google Sheets, and Email

https://n8nworkflows.xyz/workflows/automated-brand-mentions-tracker-with-gpt-4o--google-sheets--and-email-4443


# Automated Brand Mentions Tracker With GPT-4o, Google Sheets, and Email

### 1. Workflow Overview

This workflow automates the tracking of brand mentions by querying OpenAI’s GPT-4o model with a predefined set of brand-related questions. It processes the AI responses to detect whether the brand is mentioned, logs all results into Google Sheets, and sends a daily email report summarizing findings including competitor mentions.

**Target Use Cases:**  
- Monitoring brand reputation and awareness through AI-generated insights.  
- Tracking competitor mentions alongside brand mentions.  
- Automating daily reporting for marketing or brand management teams.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow automatically on a daily basis (configurable).  
- **1.2 Query Preparation:** Generates a list of brand-related queries dynamically based on user configuration.  
- **1.3 AI Interaction:** Sends each query to OpenAI’s GPT-4o API and retrieves responses.  
- **1.4 Response Processing:** Parses and analyzes responses to detect brand mentions and errors.  
- **1.5 Data Storage:** Appends processed results into a Google Sheets document.  
- **1.6 Reporting:** Summarizes the data into an HTML email, including competitor mentions, and sends it via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
Triggers the entire workflow automatically at a set interval (daily by default).

- **Nodes Involved:**  
  - Daily Trigger

- **Node Details:**  
  - **Daily Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Default daily trigger with no time restrictions (can be configured to other intervals).  
    - Input/Output: No input, outputs a trigger event to "Setup Queries".  
    - Edge Cases: Misconfiguration may cause multiple or no triggers; downtime of n8n instance will delay execution.

---

#### 2.2 Query Preparation

- **Overview:**  
Defines and outputs a list of brand-related queries for AI processing. Users customize brand name and queries in JavaScript.

- **Nodes Involved:**  
  - Setup Queries

- **Node Details:**  
  - **Setup Queries**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Brand name and list of queries hardcoded in JS for easy customization. Queries cover brand awareness, problem-solving, competitor comparisons, and more.  
      - Outputs an array of items, each containing a single query with metadata (brand name, query index, timestamp, date).  
    - Key Variables: `brandName`, `queries` array, `items` output.  
    - Input/Output: Input from Scheduler, output array to "Query ChatGPT (HTTP)".  
    - Edge Cases: If brand name or queries are empty or incorrectly formatted, downstream nodes may fail to detect brand mentions.  
    - Version: Requires n8n v2+ to support code node v2.

---

#### 2.3 AI Interaction

- **Overview:**  
Sends each brand-related query individually to OpenAI’s GPT-4o API and receives a text completion response.

- **Nodes Involved:**  
  - Query ChatGPT (HTTP)

- **Node Details:**  
  - **Query ChatGPT (HTTP)**  
    - Type: HTTP Request  
    - Configuration:  
      - POST request to `https://api.openai.com/v1/chat/completions` using `chatgpt-4o-latest` model.  
      - JSON body dynamically set with current query text.  
      - Headers include `Content-Type: application/json` and OpenAI API key credential.  
      - Response fully captured to handle errors gracefully.  
    - Key Expressions: Query text from `$json.query`.  
    - Input/Output: Input from "Setup Queries", output to "Process Response".  
    - Credential: Requires an OpenAI API key configured in n8n credentials.  
    - Edge Cases: API rate limits, network errors, invalid API key, model deprecation. Response may contain errors or incomplete data.

---

#### 2.4 Response Processing

- **Overview:**  
Processes each AI response to determine if the brand is mentioned. Handles errors and truncates response text.

- **Nodes Involved:**  
  - Process Response

- **Node Details:**  
  - **Process Response**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Runs once per AI response item.  
      - Extracts original query data by index.  
      - Checks HTTP status codes and OpenAI error messages.  
      - Detects brand mention by case-insensitive substring search.  
      - Limits response string to first 500 characters for storage.  
      - Outputs combined metadata: timestamp, query, brandName, response, brandMentioned status, error messages.  
    - Key Variables: `httpResponse`, `brandName`, `response`, `brandMentioned`, `error`.  
    - Input/Output: Input from "Query ChatGPT (HTTP)", output to "Save to Google Sheets".  
    - Edge Cases: Missing or malformed API response, brand name missing, false positives/negatives in brand mention detection.

---

#### 2.5 Data Storage

- **Overview:**  
Appends processed query results and AI responses into a Google Sheets document for record keeping.

- **Nodes Involved:**  
  - Save to Google Sheets

- **Node Details:**  
  - **Save to Google Sheets**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append rows to specified Sheet ("Sheet1").  
      - Maps fields: Date, Error, Query, Response, Timestamp, Brand_Name, Query_Index, Brand_Mentioned.  
      - Document ID must be set to target Google Sheet.  
    - Credential: Requires Google Sheets OAuth2 credential with Drive API enabled.  
    - Input/Output: Input from "Process Response", output to "Generate Email Report".  
    - Edge Cases: API quota limits, sheet permission errors, invalid document ID, schema mismatch.

---

#### 2.6 Reporting

- **Overview:**  
Compiles a daily email report summarizing brand mentions, competitor mentions, and errors; sends the report via SMTP email.

- **Nodes Involved:**  
  - Generate Email Report  
  - Send Email

- **Node Details:**  
  - **Generate Email Report**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Aggregates all processed items to count brand mentions, errors, total queries.  
      - Detects mentions of predefined competitors in AI responses.  
      - Builds an HTML email with colored rows: green for brand mentions, red for errors, white otherwise.  
      - Creates competitor summary with mention counts.  
      - Constructs subject line dynamically based on mentions found.  
      - Outputs email subject and HTML content.  
    - Key Variables: `competitors` array, `findCompetitors` function, counts and summary variables.  
    - Input/Output: Input from "Save to Google Sheets", output to "Send Email".  
    - Edge Cases: No processed data available, empty competitor list, malformed response strings.

  - **Send Email**  
    - Type: Email Send  
    - Configuration:  
      - Sends email using SMTP credentials.  
      - Subject and HTML body from previous node.  
      - Requires valid "toEmail" and "fromEmail" addresses.  
    - Credential: SMTP account (supports Gmail SMTP).  
    - Input/Output: Input from "Generate Email Report".  
    - Edge Cases: SMTP connection errors, invalid email addresses, email blocked by spam filters.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                        | Input Node(s)        | Output Node(s)          | Sticky Note                                                                                         |
|---------------------|--------------------|-------------------------------------|----------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| Daily Trigger       | Schedule Trigger    | Initiates workflow daily             | —                    | Setup Queries           | Choose your timeframe. How often do you want the automation to trigger?                           |
| Setup Queries       | Code (JavaScript)  | Creates brand-related queries array  | Daily Trigger        | Query ChatGPT (HTTP)    | Set up your brand name in the JavaScript + add the queries you want to track                      |
| Query ChatGPT (HTTP) | HTTP Request       | Sends each query to OpenAI GPT-4o    | Setup Queries        | Process Response        | Sending the queries to ChatGPT. I recommend using the 'chatgpt-4o-latest' model. You need to add your API key. |
| Process Response    | Code (JavaScript)  | Analyzes AI responses for brand mention | Query ChatGPT (HTTP) | Save to Google Sheets   | This JavaScript node analyzes each response received from ChatGPT.                               |
| Save to Google Sheets | Google Sheets      | Logs processed data to spreadsheet   | Process Response     | Generate Email Report   | Connect your Google Sheets. Don't forget to enable the Google Sheets and Google Drive APIs in your Google Cloud project. |
| Generate Email Report | Code (JavaScript)  | Generates HTML email summary          | Save to Google Sheets | Send Email              | This node turns all the raw data into a nicely formated email. No need to change anything.        |
| Send Email          | Email Send         | Sends the email report                | Generate Email Report | —                       | Finally! Replace the placeholder emails and connect yours. Using Gmail? Leave smtp.gmail.com as host. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Daily Trigger`  
   - Set to trigger daily (or preferred interval).  
   - Connect output to `Setup Queries`.

3. **Add a Code node:**  
   - Name: `Setup Queries`  
   - Paste the provided JavaScript code that defines the brand name and queries array.  
   - Customize `brandName` string to your actual brand.  
   - Customize queries array to reflect relevant queries for your brand and industry.  
   - Connect output to `Query ChatGPT (HTTP)`.

4. **Add an HTTP Request node:**  
   - Name: `Query ChatGPT (HTTP)`  
   - Method: POST  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Authentication: Use OpenAI API credentials (OAuth or API Key).  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "model": "chatgpt-4o-latest",
       "messages": [
         {
           "role": "user",
           "content": "={{ $json.query }}"
         }
       ],
       "max_tokens": 1000,
       "temperature": 0.7
     }
     ```  
   - Connect output to `Process Response`.

5. **Add a Code node:**  
   - Name: `Process Response`  
   - Paste the provided JavaScript code to parse response, detect brand mentions, handle errors, and truncate responses.  
   - Connect output to `Save to Google Sheets`.

6. **Add a Google Sheets node:**  
   - Name: `Save to Google Sheets`  
   - Operation: Append  
   - Document ID: Set your Google Sheet’s ID (ensure you have enabled Google Sheets and Drive APIs in your Google Cloud Console).  
   - Sheet Name: `Sheet1` (or your preferred sheet tab).  
   - Map columns according to the keys: Date, Error, Query, Response, Timestamp, Brand_Name, Query_Index, Brand_Mentioned.  
   - Authenticate using Google OAuth2 credentials with required scopes.  
   - Connect output to `Generate Email Report`.

7. **Add a Code node:**  
   - Name: `Generate Email Report`  
   - Paste the provided JavaScript code that aggregates results, detects competitor mentions, builds an HTML summary email, and creates a dynamic subject.  
   - Connect output to `Send Email`.

8. **Add an Email Send node:**  
   - Name: `Send Email`  
   - Set `To Email` and `From Email` with your actual email addresses.  
   - Use SMTP credentials (e.g., Gmail SMTP with OAuth or username/password).  
   - Map subject and HTML body from the previous node’s output.  
   - Connect input from `Generate Email Report`.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Choose your timeframe. How often do you want the automation to trigger?                       | Sticky Note on `Daily Trigger` node                                                                                |
| Set up your brand name in the JavaScript + add the queries you want to track                  | Sticky Note on `Setup Queries` node                                                                                 |
| Sending the queries to ChatGPT. I recommend using the 'chatgpt-4o-latest' model. You need to add your API key. | Sticky Note on `Query ChatGPT (HTTP)` node                                                                         |
| This JavaScript node analyzes each response received from ChatGPT.                           | Sticky Note on `Process Response` node                                                                              |
| Connect your Google Sheets. Don't forget to enable the Google Sheets and Google Drive APIs in your Google Cloud project. | Sticky Note on `Save to Google Sheets` node                                                                         |
| This node turns all the raw data into a nicely formated email. No need to change anything.    | Sticky Note on `Generate Email Report` node                                                                         |
| Finally! Replace the placeholder emails and connect yours. Using Gmail? Leave smtp.gmail.com as host. | Sticky Note on `Send Email` node                                                                                     |

---

**Disclaimer:**  
The provided text and workflow are generated from an automated n8n workflow. All content complies with current content policies and contains no illegal, offensive, or protected materials. All data processed is legal and public.