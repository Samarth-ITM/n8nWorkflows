Predict Restaurant Food Waste with Gemini AI and Google Sheets Reporting

https://n8nworkflows.xyz/workflows/predict-restaurant-food-waste-with-gemini-ai-and-google-sheets-reporting-5982


# Predict Restaurant Food Waste with Gemini AI and Google Sheets Reporting

### 1. Workflow Overview

This workflow automates the daily prediction of restaurant food demand and waste reduction by leveraging historical sales data and AI forecasting (Google Gemini). It is designed for restaurant managers and kitchen staff to optimize inventory, minimize food waste, and improve operational efficiency. The workflow consists of the following logical blocks:

- **1.1 Scheduled Input Trigger:** Initiates the workflow once daily at a specified hour.
- **1.2 Data Retrieval:** Fetches historical sales and raw material usage data from a Google Sheets document.
- **1.3 Data Preparation:** Cleans and structures the fetched data to a format suited for AI forecasting.
- **1.4 AI Forecasting:** Uses Google Gemini AI to analyze past trends and generate next-day predictions on sales, raw material use, and expected waste reduction.
- **1.5 AI Output Processing:** Parses and cleans the AI-generated JSON forecast data.
- **1.6 Data Storage:** Appends the processed forecast data back into a dedicated Google Sheets document for records.
- **1.7 Email Summary Generation:** Creates a clear, professional email summary of the forecast using AI.
- **1.8 Email Delivery:** Sends the forecast summary via email to designated recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Trigger

- **Overview:**  
  Triggers the workflow daily at 22:00 hours to start the data processing and prediction cycle.

- **Nodes Involved:**  
  - Daily Trigger

- **Node Details:**  
  - **Daily Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs once daily at 22:00 (10 PM) local time.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Fetch Historical Sales Data" node.  
    - Failure Modes: Scheduling misconfiguration or system clock issues might prevent triggering.  
    - Notes: Initiates the workflow every day for food waste prediction.

#### 2.2 Data Retrieval

- **Overview:**  
  Reads historical sales and raw material usage data from Google Sheets to provide the AI with context for forecasting.

- **Nodes Involved:**  
  - Fetch Historical Sales Data

- **Node Details:**  
  - **Fetch Historical Sales Data**  
    - Type: Google Sheets  
    - Configuration:  
      - Document ID: Points to the restaurant’s "Restaurant stock predictions" Google Sheets document.  
      - Sheet Name: Targets the "food wastage data" sheet.  
      - Authentication: Uses a Google Service Account credential.  
    - Inputs: Triggered by "Daily Trigger".  
    - Outputs: Sends raw rows of data to "Format Data for AI Forecasting".  
    - Failure Modes: Authentication errors, missing permissions, sheet not found, or network issues.  
    - Notes: Reads past food usage & sales data.

#### 2.3 Data Preparation

- **Overview:**  
  Transforms raw spreadsheet rows into a single structured JSON object for AI consumption.

- **Nodes Involved:**  
  - Format Data for AI Forecasting

- **Node Details:**  
  - **Format Data for AI Forecasting**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Aggregates incoming rows into one JSON object under the field `data.rows`.  
      - Returns a single output item with the entire dataset bundled.  
    - Inputs: From "Fetch Historical Sales Data".  
    - Outputs: To "AI Forecast Generator".  
    - Failure Modes: Could fail if input data is empty or malformed.  
    - Notes: Cleans and organizes raw data for AI processing.

#### 2.4 AI Forecasting

- **Overview:**  
  Employs Google Gemini AI to analyze historical data and produce next-day sales and raw material usage forecasts with waste reduction estimates.

- **Nodes Involved:**  
  - AI Forecast Generator

- **Node Details:**  
  - **AI Forecast Generator**  
    - Type: LangChain Agent (AI)  
    - Configuration:  
      - Input: JSON data from previous node.  
      - System Message: Detailed instructions for forecasting demand and waste reduction per dish and raw material, including output JSON schema and constraints.  
      - Model: Gemini AI via LangChain agent interface.  
    - Inputs: Receives structured data from "Format Data for AI Forecasting".  
    - Outputs: Raw AI JSON output to "Clean & Structure AI Output".  
    - Failure Modes: API rate limits, AI model downtime, malformed AI responses, or incomplete data.  
    - Notes: Uses Gemini AI to forecast demand and recommend waste reduction.

#### 2.5 AI Output Processing

- **Overview:**  
  Parses the raw AI JSON text output into structured JSON objects for further processing and storage.

- **Nodes Involved:**  
  - Clean & Structure AI Output

- **Node Details:**  
  - **Clean & Structure AI Output**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Removes code block formatting and prefixes from AI response.  
      - Safely parses the cleaned string into JSON array.  
      - Outputs each forecast entry as a separate item.  
    - Inputs: Raw AI string output from "AI Forecast Generator".  
    - Outputs: Structured JSON items to "Log Forecast to Google Sheets".  
    - Failure Modes: Parsing errors if AI output is malformed or incomplete. Throws explicit error with debug info on failure.  
    - Notes: Parses AI response into structured and clean format for reporting.

#### 2.6 Data Storage

- **Overview:**  
  Appends the clean, forecasted data into a dedicated Google Sheets sheet for record-keeping and further analysis.

- **Nodes Involved:**  
  - Log Forecast to Google Sheets

- **Node Details:**  
  - **Log Forecast to Google Sheets**  
    - Type: Google Sheets  
    - Configuration:  
      - Document ID: Same as input document.  
      - Sheet Name: "predicted food data for low wastage".  
      - Operation: Append rows.  
      - Column Mapping: Automatically maps incoming fields to sheet columns.  
      - Authentication: Google Service Account.  
    - Inputs: Structured forecast items from "Clean & Structure AI Output".  
    - Outputs: To "Create Email Summary".  
    - Failure Modes: Authentication errors, permission issues, sheet not found, API quota limits.  
    - Notes: Stores AI-generated forecast back into a forecast-specific Google Sheet.

#### 2.7 Email Summary Generation

- **Overview:**  
  Uses AI to draft a professional, human-friendly email summarizing the forecast results and suggesting actions.

- **Nodes Involved:**  
  - Create Email Summary

- **Node Details:**  
  - **Create Email Summary**  
    - Type: LangChain Agent (AI)  
    - Configuration:  
      - Input: Raw AI forecast output JSON from "AI Forecast Generator".  
      - System Message: Instructions to produce a concise, warm, professional summary email with bullet points per dish, predicted usage, and waste reduction.  
      - Model: Gemini AI via LangChain agent interface.  
      - Executes once per workflow run.  
    - Inputs: Forecast JSON array from AI output node.  
    - Outputs: Email body text to "Send Email Forecast Report".  
    - Failure Modes: API issues, malformed input data, or generation errors.  
    - Notes: Creates a concise, human-friendly summary of the forecast.

#### 2.8 Email Delivery

- **Overview:**  
  Sends the generated forecast summary email to designated recipients (kitchen manager, staff).

- **Nodes Involved:**  
  - Send Email Forecast Report

- **Node Details:**  
  - **Send Email Forecast Report**  
    - Type: Gmail  
    - Configuration:  
      - Recipient: abc@gmail.com (placeholder, should be replaced with real address).  
      - Subject: "Next monday prediction".  
      - Email Body: Uses output text from "Create Email Summary".  
      - Email Type: Plain text.  
      - Credential: Gmail OAuth2 for authentication.  
    - Inputs: Email body text from "Create Email Summary".  
    - Outputs: None (final node).  
    - Failure Modes: Authentication failure, SMTP errors, invalid recipient address.  
    - Notes: Delivers the forecast report via email to decision-makers.

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                            | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                   |
|---------------------------|--------------------------------|--------------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Daily Trigger             | Schedule Trigger                | Initiates workflow daily trigger             | None                         | Fetch Historical Sales Data   | Initiates the workflow every day to perform food waste prediction.                           |
| Fetch Historical Sales Data | Google Sheets                  | Reads past food usage & sales data           | Daily Trigger                | Format Data for AI Forecasting | Reads past food usage & sales from Google Sheets to understand trends.                       |
| Format Data for AI Forecasting | Code                         | Cleans and formats raw data for AI            | Fetch Historical Sales Data  | AI Forecast Generator         | Cleans and organizes raw data into a structure suitable for AI processing.                   |
| AI Forecast Generator      | LangChain Agent (AI)            | Generates next-day sales and waste forecast  | Format Data for AI Forecasting | Clean & Structure AI Output  | Uses Gemini AI to forecast food demand and recommend waste reduction.                        |
| Clean & Structure AI Output | Code                          | Parses and cleans AI JSON output               | AI Forecast Generator        | Log Forecast to Google Sheets | Parses AI response into structured and clean format for reporting.                          |
| Log Forecast to Google Sheets | Google Sheets                 | Stores forecast data back into Google Sheets  | Clean & Structure AI Output  | Create Email Summary          | Stores AI-generated forecast back into a forecast-specific Google Sheet.                    |
| Create Email Summary       | LangChain Agent (AI)            | Drafts professional email summary of forecast | Log Forecast to Google Sheets | Send Email Forecast Report    | Creates a concise, human-friendly summary of the forecast.                                  |
| Send Email Forecast Report | Gmail                          | Sends forecast summary email                   | Create Email Summary         | None                         | Delivers the forecast report via email to decision-makers (kitchen, manager, etc).          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Daily Trigger" node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 22:00 (10 PM).

2. **Add "Fetch Historical Sales Data" node:**  
   - Type: Google Sheets  
   - Document ID: Use your Google Sheets document ID (e.g., restaurant stock predictions).  
   - Sheet Name: Select the sheet containing historical food wastage data.  
   - Authentication: Use a Google Service Account credential with read access.  
   - Connect "Daily Trigger" output to this node’s input.

3. **Create "Format Data for AI Forecasting" node:**  
   - Type: Code (JavaScript)  
   - Paste the following JS code to aggregate rows:
   ```js
   const items = $input.all();
   const rawRows = items.map(item => item.json);
   const payload = { rows: rawRows };
   return [{ json: { data: payload } }];
   ```
   - Connect "Fetch Historical Sales Data" output to this node.

4. **Add "AI Forecast Generator" node:**  
   - Type: LangChain Agent  
   - Configure system message with detailed AI instructions for forecasting demand and waste reduction (as described in section 2.4).  
   - Input: Use expression to pass data: `={{ $json.data }}`  
   - Set AI model to Google Gemini (or Gemini 2.5 Pro).  
   - Connect "Format Data for AI Forecasting" output here.

5. **Create "Clean & Structure AI Output" node:**  
   - Type: Code (JavaScript)  
   - Paste the code to clean AI JSON output:
   ```js
   const raw = $input.item.json.output || $input.item.json.content;
   let cleaned = raw.replace(/```json/, '').replace(/```/g, '').replace(/^(Here is (the )?JSON[:\s]*)/, '').trim();
   let parsed;
   try {
     parsed = JSON.parse(cleaned);
   } catch (e) {
     throw new Error('Failed to parse AI JSON: ' + e.message + '\nRaw cleaned text: ' + cleaned);
   }
   return parsed.map(obj => ({ json: obj }));
   ```
   - Connect "AI Forecast Generator" output here.

6. **Add "Log Forecast to Google Sheets" node:**  
   - Type: Google Sheets  
   - Document ID: Same as the source document.  
   - Sheet Name: Create or select a sheet for forecast data logging (e.g., "predicted food data for low wastage").  
   - Operation: Append rows.  
   - Map columns automatically to forecast output fields.  
   - Authentication: Use Google Service Account.  
   - Connect "Clean & Structure AI Output" output here.

7. **Add "Create Email Summary" node:**  
   - Type: LangChain Agent  
   - Configure system message to generate a friendly, professional email summary based on forecast JSON array (see section 2.7 for instructions).  
   - Input: Use expression to pass the raw output JSON from the AI forecast node (not from cleaned output), e.g., `={{ $('AI Forecast Generator').item.json.output }}`  
   - Connect "Log Forecast to Google Sheets" output here.

8. **Add "Send Email Forecast Report" node:**  
   - Type: Gmail  
   - Set recipient email (replace placeholder with actual).  
   - Subject: "Next monday prediction" or appropriate subject.  
   - Message: Use expression `={{ $json.output }}` to send the email body generated in the previous node.  
   - Configure Gmail OAuth2 credentials.  
   - Connect "Create Email Summary" output here.

9. **Validate all connections:**  
   - Daily Trigger → Fetch Historical Sales Data → Format Data for AI Forecasting → AI Forecast Generator → Clean & Structure AI Output → Log Forecast to Google Sheets → Create Email Summary → Send Email Forecast Report.

10. **Test the workflow:**  
    - Manually trigger the workflow to verify data flows correctly, AI calls succeed, and emails are sent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow Purpose: Restaurant Food Waste Prediction System automates daily forecasting of sales and raw material needs to minimize waste. | Sticky Note8 in the workflow overview area                                                        |
| Gemini AI model used is "models/gemini-2.5-pro" via LangChain integration for natural language processing and forecasting tasks.       | Node "AI Forecast Generator" and "Create Email Summary"                                           |
| Google Sheets Service Account credentials require appropriate permissions for reading/writing to the target spreadsheet.                | Nodes "Fetch Historical Sales Data" and "Log Forecast to Google Sheets"                           |
| Gmail OAuth2 credentials must be set up with proper scopes to send emails on behalf of the user.                                         | Node "Send Email Forecast Report"                                                                 |
| Example AI output format and email summary style are specified in system messages to ensure consistent results.                         | Nodes "AI Forecast Generator" and "Create Email Summary"                                          |
| The workflow is designed to run automatically daily without manual intervention, improving operational efficiency.                       | General workflow design note                                                                      |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow, respecting all content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.