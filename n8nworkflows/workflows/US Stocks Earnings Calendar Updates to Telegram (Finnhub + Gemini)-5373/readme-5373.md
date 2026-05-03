US Stocks Earnings Calendar Updates to Telegram (Finnhub + Gemini)

https://n8nworkflows.xyz/workflows/us-stocks-earnings-calendar-updates-to-telegram--finnhub---gemini--5373


# US Stocks Earnings Calendar Updates to Telegram (Finnhub + Gemini)

### 1. Workflow Overview

This workflow automates the retrieval and dissemination of upcoming US stock earnings calendar updates using financial data from Finnhub and formats the information via Google Gemini AI before sending it as a Telegram message. Scheduled to run every 3 days, it dynamically sets the date range for earnings data, fetches and organizes the data, leverages AI to format and parse it, and finally sends the update to a Telegram channel or user.

**Logical blocks:**

- **1.1 Schedule Trigger & Date Configuration:** Automates periodic execution and dynamically determines the relevant date range for earnings data retrieval.
- **1.2 API Key Setup & Data Retrieval:** Prepares API credentials and queries the Finnhub API to get upcoming earnings.
- **1.3 Data Organization:** Processes raw earnings data into a structured format suitable for AI input.
- **1.4 AI Processing & Formatting:** Uses Google Gemini language model and LangChain AI agent to format and parse the earnings data into a clean, human-readable message.
- **1.5 Notification Dispatch:** Sends the formatted earnings update to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger & Date Configuration

**Overview:**  
This block triggers the workflow every three days and dynamically calculates the date range for querying upcoming earnings.

**Nodes Involved:**  
- Schedule Every 3 Days  
- Dynamically Sets the Date  

**Node Details:**

- **Schedule Every 3 Days**  
  - *Type & Role:* Schedule Trigger; initiates workflow execution on a fixed interval (every 3 days).  
  - *Configuration:* Default schedule node, set to trigger every 3 days.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Sends trigger signal to "Dynamically Sets the Date".  
  - *Potential Failures:* Scheduling misconfiguration or node downtime (rare).  

- **Dynamically Sets the Date**  
  - *Type & Role:* Code node; calculates dynamic date range (e.g., today's date and a future date) to query earnings data within this interval.  
  - *Configuration:* JavaScript code that likely computes start and end dates based on current date.  
  - *Inputs:* Trigger signal from Schedule node.  
  - *Outputs:* Passes computed dates to "Set API Key for Finhubb & Dates".  
  - *Edge Cases:* Date calculation errors, timezone issues, or invalid date formats.  

---

#### 2.2 API Key Setup & Data Retrieval

**Overview:**  
Sets the required API key and other parameters, then queries Finnhub API to retrieve upcoming earnings within the computed date range.

**Nodes Involved:**  
- Set API Key for Finhubb & Dates  
- Gets Upcoming Earnings  

**Node Details:**

- **Set API Key for Finhubb & Dates**  
  - *Type & Role:* Set node; injects API key for Finnhub and likely adds the date parameters to the data payload.  
  - *Configuration:* Variables set for Finnhub API key and date parameters passed from previous node.  
  - *Inputs:* Date data from "Dynamically Sets the Date".  
  - *Outputs:* Passes credentials and parameters to "Gets Upcoming Earnings".  
  - *Edge Cases:* Missing or invalid API key, incorrect parameter names causing API call failure.  

- **Gets Upcoming Earnings**  
  - *Type & Role:* HTTP Request node; calls Finnhub API endpoint to fetch earnings calendar data.  
  - *Configuration:*  
    - HTTP method: GET (likely)  
    - URL: Finnhub earnings calendar endpoint  
    - Authentication: Uses API key set earlier  
    - Query parameters: Date range for earnings  
  - *Inputs:* From "Set API Key for Finhubb & Dates".  
  - *Outputs:* Raw earnings data (JSON) to "Organizes Input".  
  - *Edge Cases:* API rate limits, network errors, invalid API key, empty or malformed responses, endpoint downtime.  

---

#### 2.3 Data Organization

**Overview:**  
Processes raw earnings data to a structured format suitable for AI consumption.

**Nodes Involved:**  
- Organizes Input  

**Node Details:**

- **Organizes Input**  
  - *Type & Role:* Code node; cleans, filters, or transforms the API data into a summarized or structured format.  
  - *Configuration:* JavaScript code likely extracts relevant fields (e.g., company name, earnings date, EPS estimates).  
  - *Inputs:* Raw data from "Gets Upcoming Earnings".  
  - *Outputs:* Structured data passed to "AI Agent".  
  - *Edge Cases:* Data inconsistencies, empty datasets, unexpected API response structure causing code errors.  

---

#### 2.4 AI Processing & Formatting

**Overview:**  
Uses Google Gemini LLM and LangChain AI Agent to convert structured earnings data into a human-readable message formatted for Telegram.

**Nodes Involved:**  
- Google Gemini Chat Model (Formats Output)  
- Structured Output Parser  
- AI Agent  

**Node Details:**

- **Google Gemini Chat Model (Formats Output)**  
  - *Type & Role:* LangChain AI language model node using Google Gemini; generates a formatted text output from structured earnings data.  
  - *Configuration:* Model parameters configured for chat-based generation, possibly including prompt templates or system instructions.  
  - *Inputs:* Structured data from "Organizes Input".  
  - *Outputs:* Formatted text to "AI Agent".  
  - *Edge Cases:* API quota limits, response timeouts, malformed prompt causing poor output.  

- **Structured Output Parser**  
  - *Type & Role:* LangChain structured output parser; parses AI-generated content to ensure consistent output format for downstream nodes.  
  - *Configuration:* Parsing schema defined for expected output (e.g., JSON or structured text).  
  - *Inputs:* AI output from "AI Agent".  
  - *Outputs:* Parsed, validated content back to "AI Agent" for further processing or validation.  
  - *Edge Cases:* Parsing failures if AI output deviates from expected schema.  

- **AI Agent**  
  - *Type & Role:* LangChain AI Agent; orchestrates the interaction between the language model and output parser, managing prompts and responses to produce final content.  
  - *Configuration:* Connected to both "Google Gemini Chat Model" (as language model) and "Structured Output Parser".  
  - *Inputs:* Structured input from "Organizes Input" and parsed output from "Structured Output Parser".  
  - *Outputs:* Final formatted message text to "Send Upcoming Earning Updates via Telegram".  
  - *Edge Cases:* Integration failures, prompt mismanagement, AI model unavailability.  

---

#### 2.5 Notification Dispatch

**Overview:**  
Sends the AI-formatted earnings update message to a Telegram chat or channel.

**Nodes Involved:**  
- Send Upcoming Earning Updates via Telegram  

**Node Details:**

- **Send Upcoming Earning Updates via Telegram**  
  - *Type & Role:* Telegram node; sends a message to a configured Telegram chat using bot credentials.  
  - *Configuration:*  
    - Telegram Bot API credentials (OAuth2 or Bot Token)  
    - Chat ID or channel ID where message is sent  
    - Message content from AI Agent output  
  - *Inputs:* Final message from "AI Agent".  
  - *Outputs:* None (terminal node).  
  - *Edge Cases:* Telegram API errors, invalid chat ID, bot permissions issues, message length limits, network errors.  

---

### 3. Summary Table

| Node Name                             | Node Type                           | Functional Role                      | Input Node(s)                    | Output Node(s)                                   | Sticky Note |
|-------------------------------------|-----------------------------------|------------------------------------|---------------------------------|-------------------------------------------------|-------------|
| Schedule Every 3 Days                | Schedule Trigger                  | Initiates workflow every 3 days    | None                            | Dynamically Sets the Date                         |             |
| Dynamically Sets the Date            | Code                             | Calculates dynamic date range      | Schedule Every 3 Days            | Set API Key for Finhubb & Dates                  |             |
| Set API Key for Finhubb & Dates     | Set                              | Sets API key and date parameters   | Dynamically Sets the Date        | Gets Upcoming Earnings                           |             |
| Gets Upcoming Earnings              | HTTP Request                     | Fetches earnings data from Finnhub | Set API Key for Finhubb & Dates | Organizes Input                                  |             |
| Organizes Input                     | Code                             | Structures raw earnings data       | Gets Upcoming Earnings          | AI Agent                                         |             |
| AI Agent                           | LangChain AI Agent               | Orchestrates AI processing         | Organizes Input, Structured Output Parser, Google Gemini Chat Model | Send Upcoming Earning Updates via Telegram |             |
| Google Gemini Chat Model (Formats Output) | LangChain Language Model Node | Formats earnings data via AI       | Organizes Input                 | AI Agent                                         |             |
| Structured Output Parser           | LangChain Output Parser          | Parses AI output for structure     | AI Agent (ai_outputParser)      | AI Agent                                         |             |
| Send Upcoming Earning Updates via Telegram | Telegram Node                  | Sends formatted message to Telegram | AI Agent                       | None                                            |             |
| Sticky Note                        | Sticky Note                     | Annotation                        | None                            | None                                            |             |
| Sticky Note1                       | Sticky Note                     | Annotation                        | None                            | None                                            |             |
| Sticky Note2                       | Sticky Note                     | Annotation                        | None                            | None                                            |             |
| Sticky Note3                       | Sticky Note                     | Annotation                        | None                            | None                                            |             |
| Sticky Note4                       | Sticky Note                     | Annotation                        | None                            | None                                            |             |
| Sticky Note5                       | Sticky Note                     | Annotation                        | None                            | None                                            |             |

*Note: All sticky notes are empty and contain no comments.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: "Schedule Every 3 Days"  
   - Type: Schedule Trigger  
   - Configure to trigger every 3 days (e.g., set interval to 72 hours).  

2. **Create a Code Node to Calculate Dates**  
   - Name: "Dynamically Sets the Date"  
   - Type: Code  
   - Use JavaScript to compute start and end dates (e.g., today’s date and 3 days ahead).  
   - Connect input from "Schedule Every 3 Days".  
   - Output date parameters (e.g., start_date, end_date) to next node.  

3. **Create a Set Node to Add API Key and Dates**  
   - Name: "Set API Key for Finhubb & Dates"  
   - Type: Set  
   - Add fields:  
     - `finnhub_api_key` (your Finnhub API key credential)  
     - `start_date` and `end_date` from previous code node output  
   - Connect input from "Dynamically Sets the Date".  

4. **Create an HTTP Request Node to Fetch Earnings**  
   - Name: "Gets Upcoming Earnings"  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Finnhub earnings calendar endpoint (e.g., `https://finnhub.io/api/v1/calendar/earnings`)  
   - Add query parameters using the set data: `from=start_date`, `to=end_date`, and API key.  
   - Connect input from "Set API Key for Finhubb & Dates".  

5. **Create a Code Node to Organize Input Data**  
   - Name: "Organizes Input"  
   - Type: Code  
   - Write JavaScript to parse the raw JSON from Finnhub and extract relevant info: company, symbol, earnings date/time, EPS estimates, etc.  
   - Format data as structured JSON or text for AI consumption.  
   - Connect input from "Gets Upcoming Earnings".  

6. **Create a Google Gemini Chat Model Node**  
   - Name: "Google Gemini Chat Model (Formats Output)"  
   - Type: LangChain Language Model (Google Gemini)  
   - Configure with your Google Gemini API credentials.  
   - Set prompt or system instructions to convert structured earnings data into a user-friendly text message.  
   - Connect input from "Organizes Input".  

7. **Create a LangChain Structured Output Parser Node**  
   - Name: "Structured Output Parser"  
   - Type: LangChain Output Parser  
   - Define parsing schema to validate AI output structure (e.g., JSON with fields: message, summary, etc.).  
   - Connect input from "AI Agent" (ai_outputParser port).  

8. **Create a LangChain AI Agent Node**  
   - Name: "AI Agent"  
   - Type: LangChain AI Agent  
   - Link "Google Gemini Chat Model" node as the language model input.  
   - Link "Structured Output Parser" node as the output parser.  
   - Connect input from "Organizes Input".  
   - Connect output to Telegram node next.  

9. **Create Telegram Node to Send Message**  
   - Name: "Send Upcoming Earning Updates via Telegram"  
   - Type: Telegram  
   - Configure Telegram Bot API credentials (Bot Token or OAuth2).  
   - Set chat ID or channel ID to send messages.  
   - Use the output of "AI Agent" as message content.  
   - Connect input from "AI Agent".  

10. **Connect All Nodes in Order:**  
    Schedule Trigger → Dynamically Sets the Date → Set API Key for Finhubb & Dates → Gets Upcoming Earnings → Organizes Input → Google Gemini Chat Model → AI Agent → Structured Output Parser (ai_outputParser)  
    AI Agent (main output) → Send Upcoming Earning Updates via Telegram  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Finnhub API documentation is essential for correct endpoint usage and parameter setup.                       | https://finnhub.io/docs/api#earnings-calendar                                                                 |
| Google Gemini LLM usage requires proper API credential setup and quota management.                           | https://developers.google.com/ Gemini (confirm exact URL)                                                     |
| Telegram Bot setup requires creating a bot through BotFather and adding it to desired chat/channel.          | https://core.telegram.org/bots#6-botfather                                                                    |
| LangChain nodes enable advanced AI orchestration and require n8n integration with LangChain package.         | https://docs.n8n.io/integrations/builtin/nodes/ai/                                                            |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.