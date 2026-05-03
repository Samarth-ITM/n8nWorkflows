Get IPO Calendar Alerts via Telegram with Finnhub API and Gemini AI

https://n8nworkflows.xyz/workflows/get-ipo-calendar-alerts-via-telegram-with-finnhub-api-and-gemini-ai-5439


# Get IPO Calendar Alerts via Telegram with Finnhub API and Gemini AI

### 1. Workflow Overview

This workflow automates the retrieval and notification of upcoming IPO (Initial Public Offering) calendar events using the Finnhub API, enhanced by AI processing with Google Gemini and LangChain AI agents, and delivers alerts via Telegram. It is designed for users who want regular, intelligently formatted updates about IPO schedules without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Date Setup:** Periodically triggers the workflow every 7 days and dynamically sets the date range for the IPO query.
- **1.2 API Request & Data Retrieval:** Uses the Finnhub API to fetch IPO calendar data based on the set dates.
- **1.3 Data Organization & AI Processing:** Organizes incoming raw data, formats and parses it through AI models (Google Gemini and LangChain AI Agent) for structured output.
- **1.4 Notification Dispatch:** Sends the processed IPO calendar alerts to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Date Setup

- **Overview:**  
  This block periodically initiates the workflow on a weekly basis, dynamically computes the date parameters needed for the IPO query, and sets the API key and date variables for the Finnhub API request.

- **Nodes Involved:**  
  - Schedule Every 7 Days  
  - Dynamically Sets the Date  
  - Set API Key for Finhubb & Dates

- **Node Details:**

  - **Schedule Every 7 Days**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every 7 days automatically.  
    - Configuration: Default weekly schedule trigger (exact time not specified).  
    - Inputs: None (trigger node).  
    - Outputs: Triggers the next node "Dynamically Sets the Date".  
    - Edge cases: Misconfiguration of schedule time or timezone could cause timing issues.

  - **Dynamically Sets the Date**  
    - Type: Code Node (JavaScript)  
    - Role: Calculates the current date and possibly a range of dates for querying IPO events dynamically.  
    - Configuration: Custom JavaScript code (not provided) to produce date values.  
    - Inputs: Trigger from schedule.  
    - Outputs: Data passed to "Set API Key for Finhubb & Dates".  
    - Variables: Likely sets variables like `startDate` and `endDate`.  
    - Edge cases: Code errors or invalid date calculations could halt the workflow.

  - **Set API Key for Finhubb & Dates**  
    - Type: Set Node  
    - Role: Sets the Finnhub API key and date parameters for the HTTP request node.  
    - Configuration: Injects credentials and date parameters into the workflow context or data.  
    - Inputs: From "Dynamically Sets the Date".  
    - Outputs: Passes data to "Gets Upcoming Earnings".  
    - Edge cases: Missing or invalid API key could cause authentication errors.

---

#### 2.2 API Request & Data Retrieval

- **Overview:**  
  Performs the HTTP request to the Finnhub API to fetch IPO calendar events using the prepared dates and API key.

- **Nodes Involved:**  
  - Gets Upcoming Earnings

- **Node Details:**

  - **Gets Upcoming Earnings**  
    - Type: HTTP Request Node  
    - Role: Queries the Finnhub IPO calendar endpoint with provided dates and API key.  
    - Configuration: HTTP GET method with URL and query parameters dynamically set from previous nodes.  
    - Inputs: Receives API key and date parameters from "Set API Key for Finhubb & Dates".  
    - Outputs: Passes raw API response data to "Organizes Input".  
    - Edge cases: Network errors, API rate limits, invalid parameters, or API downtime could cause failure.

---

#### 2.3 Data Organization & AI Processing

- **Overview:**  
  Processes and organizes raw API data, then uses AI models to format the IPO information into a structured and human-readable format.

- **Nodes Involved:**  
  - Organizes Input  
  - AI Agent  
  - Google Gemini Chat Model (Formats Output)  
  - Structured Output Parser

- **Node Details:**

  - **Organizes Input**  
    - Type: Code Node (JavaScript)  
    - Role: Cleans, organizes, and prepares the raw IPO data for AI processing.  
    - Configuration: Custom JavaScript to structure or filter data before AI input.  
    - Inputs: Raw data from "Gets Upcoming Earnings".  
    - Outputs: Passes prepared data to "AI Agent".  
    - Edge cases: Data parsing errors or unexpected data formats.

  - **Google Gemini Chat Model (Formats Output)**  
    - Type: LangChain Google Gemini LM Chat Node  
    - Role: Uses Google Gemini language model to format IPO data into a readable message.  
    - Configuration: AI model parameters configured for output formatting.  
    - Inputs: Connected as AI language model input for "AI Agent".  
    - Outputs: Provides formatted text to "AI Agent".  
    - Edge cases: AI model availability, rate limits, or unexpected input format.

  - **AI Agent**  
    - Type: LangChain AI Agent Node  
    - Role: Controls AI workflow, integrates inputs, and drives output parsing.  
    - Configuration: Uses "Google Gemini Chat Model" as language model and "Structured Output Parser" for parsing AI results.  
    - Inputs: Receives organized input and AI model output.  
    - Outputs: Passes final AI-processed message to "Send Upcoming IPO Calendar Updates via Telegram".  
    - Edge cases: AI processing failures or parsing errors.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser Node  
    - Role: Parses AI-generated content into a defined structured format.  
    - Configuration: Structured parsing rules or schema (not detailed).  
    - Inputs: Linked as AI output parser for the "AI Agent".  
    - Outputs: Structured data for the AI Agent node to handle.  
    - Edge cases: Parsing mismatches or invalid AI output.

---

#### 2.4 Notification Dispatch

- **Overview:**  
  Sends the AI-formatted IPO calendar alerts via Telegram to subscribed users or groups.

- **Nodes Involved:**  
  - Send Upcoming IPO Calendar Updates via Telegram

- **Node Details:**

  - **Send Upcoming IPO Calendar Updates via Telegram**  
    - Type: Telegram Node  
    - Role: Sends the final IPO alert message to Telegram chat.  
    - Configuration: Telegram Bot credentials configured via OAuth2, webhook ID specified.  
    - Inputs: Receives formatted message from "AI Agent".  
    - Outputs: None (final node).  
    - Edge cases: Telegram API errors, invalid chat ID, or permission issues.

---

### 3. Summary Table

| Node Name                                 | Node Type                            | Functional Role                           | Input Node(s)                      | Output Node(s)                               | Sticky Note |
|-------------------------------------------|------------------------------------|-----------------------------------------|----------------------------------|----------------------------------------------|-------------|
| Schedule Every 7 Days                      | Schedule Trigger                   | Periodically triggers workflow weekly  | None                             | Dynamically Sets the Date                      |             |
| Dynamically Sets the Date                  | Code Node                         | Calculates date parameters dynamically | Schedule Every 7 Days            | Set API Key for Finhubb & Dates               |             |
| Set API Key for Finhubb & Dates            | Set Node                         | Sets API key and date parameters        | Dynamically Sets the Date        | Gets Upcoming Earnings                         |             |
| Gets Upcoming Earnings                     | HTTP Request                     | Fetches IPO calendar data from Finnhub | Set API Key for Finhubb & Dates | Organizes Input                               |             |
| Organizes Input                            | Code Node                        | Organizes and prepares raw API data     | Gets Upcoming Earnings           | AI Agent                                      |             |
| Google Gemini Chat Model (Formats Output) | LangChain LM Chat Node           | Formats IPO data via Google Gemini AI   | AI Agent (as AI language model) | AI Agent                                      |             |
| Structured Output Parser                   | LangChain Output Parser          | Parses AI output into structured format | AI Agent (as AI output parser)  | AI Agent                                      |             |
| AI Agent                                   | LangChain Agent                  | Controls AI processing workflow          | Organizes Input, Gemini Model, Parser | Send Upcoming IPO Calendar Updates via Telegram |             |
| Send Upcoming IPO Calendar Updates via Telegram | Telegram Node                    | Sends IPO alert message                   | AI Agent                       | None                                         |             |
| Sticky Note                                | Sticky Note                     | Visual comment                           | None                             | None                                         |             |
| Sticky Note1                               | Sticky Note                     | Visual comment                           | None                             | None                                         |             |
| Sticky Note2                               | Sticky Note                     | Visual comment                           | None                             | None                                         |             |
| Sticky Note3                               | Sticky Note                     | Visual comment                           | None                             | None                                         |             |
| Sticky Note4                               | Sticky Note                     | Visual comment                           | None                             | None                                         |             |
| Sticky Note5                               | Sticky Note                     | Visual comment                           | None                             | None                                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Schedule Every 7 Days"  
   - Configure to trigger every 7 days (weekly).  
   - No inputs. Output connects to next node.

2. **Add a Code Node to Dynamically Set Date**  
   - Type: Code Node (JavaScript)  
   - Name: "Dynamically Sets the Date"  
   - Write JavaScript to calculate the current date and set a date range for the IPO query (e.g., today and 7 days ahead).  
   - Connect input from "Schedule Every 7 Days".  
   - Output connects to next node.

3. **Add a Set Node to Set API Key and Dates**  
   - Type: Set Node  
   - Name: "Set API Key for Finhubb & Dates"  
   - Configure to set:  
     - Finnhub API key (stored securely in credentials or environment variables).  
     - Date parameters (`startDate`, `endDate`) passed from previous code node.  
   - Connect input from "Dynamically Sets the Date".  
   - Output connects to next node.

4. **Add an HTTP Request Node to Fetch IPO Data**  
   - Type: HTTP Request Node  
   - Name: "Gets Upcoming Earnings"  
   - Method: GET  
   - URL: Finnhub IPO calendar endpoint (e.g., `https://finnhub.io/api/v1/calendar/ipo`)  
   - Query Parameters: Include dates and API key dynamically from previous node outputs.  
   - Connect input from "Set API Key for Finhubb & Dates".  
   - Output connects to next node.

5. **Add a Code Node to Organize Input Data**  
   - Type: Code Node (JavaScript)  
   - Name: "Organizes Input"  
   - Write code to parse and filter IPO data as needed for AI processing (e.g., extract relevant fields).  
   - Connect input from "Gets Upcoming Earnings".  
   - Output connects to "AI Agent".

6. **Add a LangChain AI Agent Node**  
   - Type: LangChain AI Agent  
   - Name: "AI Agent"  
   - Configure to use:  
     - Language model: Google Gemini Chat Model node (set in next step).  
     - Output parser: Structured Output Parser node (set in step 8).  
   - Connect input from "Organizes Input".  
   - Output connects to Telegram node.

7. **Add a LangChain LM Chat Node for Google Gemini**  
   - Type: LangChain LM Chat Google Gemini  
   - Name: "Google Gemini Chat Model (Formats Output)"  
   - Configure API credentials and model parameters.  
   - Connect as AI language model input for "AI Agent".  

8. **Add a LangChain Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Name: "Structured Output Parser"  
   - Define parsing schema or rules to structure AI output.  
   - Connect as AI output parser for "AI Agent".

9. **Add a Telegram Node to Send Notifications**  
   - Type: Telegram Node  
   - Name: "Send Upcoming IPO Calendar Updates via Telegram"  
   - Configure credentials (Telegram Bot OAuth2).  
   - Set chat ID or user ID where messages will be sent.  
   - Connect input from "AI Agent".  
   - No output (terminal node).

10. **Deploy and Test the Workflow**  
    - Verify API keys are valid and permissions are set.  
    - Test schedule trigger timing and message formatting.  
    - Monitor logs for errors or edge cases.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                       |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The workflow uses Finnhub API to access real-time IPO calendar data. Ensure your API key has sufficient quota and permissions. | Finnhub API documentation: https://finnhub.io/docs/api#ipo-calendar |
| Google Gemini AI model integration requires proper LangChain node setup and valid Google Cloud credentials. | LangChain n8n nodes docs: https://docs.n8n.io/integrations/ai/       |
| Telegram node requires OAuth2 Bot credentials and chat ID configuration to send messages correctly. | Telegram Bot API: https://core.telegram.org/bots/api                  |
| Consider error handling in code nodes for date calculations and API responses to improve robustness. |                                                                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.