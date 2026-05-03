Send AI-Enhanced Economic Calendar Alerts to Telegram with Gemini-2.0-Flash

https://n8nworkflows.xyz/workflows/send-ai-enhanced-economic-calendar-alerts-to-telegram-with-gemini-2-0-flash-5450


# Send AI-Enhanced Economic Calendar Alerts to Telegram with Gemini-2.0-Flash

### 1. Workflow Overview

This workflow automates the process of fetching upcoming economic and IPO calendar events with medium to high impact, enhancing the data using AI (Google Gemini 2.0 language model), and sending summarized alerts via Telegram. It is designed for financial analysts, traders, or economic researchers who require timely, AI-enhanced alerts on economic calendar news.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Date Preparation:** Automatically triggers the workflow weekly and dynamically sets date parameters for API queries.
- **1.2 Data Retrieval:** Calls an external API (via RapidAPI) to fetch upcoming economic news/events.
- **1.3 Data Filtering:** Filters the news to retain only medium and high impact events.
- **1.4 Data Organization:** Organizes and formats the filtered data for AI processing.
- **1.5 AI Enhancement:** Uses the Google Gemini 2.0 chat model to enhance and structure the output with Langchain agent and parser.
- **1.6 Messaging:** Updates the text expression and sends the final alert message via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Date Preparation

- **Overview:**  
  This block triggers the workflow every 7 days and prepares dynamic date parameters for API requests.

- **Nodes Involved:**  
  - Schedule Every 7 Days  
  - Dynamically Sets the Date  
  - Set API Key for RapidAPI & Dates

- **Node Details:**  
  - **Schedule Every 7 Days**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow on a weekly basis.  
    - Config: Default weekly schedule without additional filters.  
    - Connections: Outputs to "Dynamically Sets the Date".  
    - Edge Cases: Missed triggers if n8n instance is down during schedule.  

  - **Dynamically Sets the Date**  
    - Type: Code (JavaScript)  
    - Role: Computes and sets dynamic date parameters (e.g., current date, date range) for API requests.  
    - Config: Custom JS code generating date strings likely in ISO format.  
    - Connections: Outputs to "Set API Key for RapidAPI & Dates".  
    - Edge Cases: Date calculation errors if system time is incorrect.  

  - **Set API Key for RapidAPI & Dates**  
    - Type: Set  
    - Role: Assigns API key credentials and combines them with date parameters for the upcoming HTTP request.  
    - Config: Stores RapidAPI key and sets parameters like start and end dates for querying economic events.  
    - Connections: Outputs to "Gets Upcoming News".  
    - Edge Cases: Missing or invalid API key causes request failures.  

#### 2.2 Data Retrieval

- **Overview:**  
  This block fetches upcoming economic/IPO news data from an external API using the configured parameters.

- **Nodes Involved:**  
  - Gets Upcoming News

- **Node Details:**  
  - **Gets Upcoming News**  
    - Type: HTTP Request  
    - Role: Queries an external API for upcoming economic calendar events with given date range and credentials.  
    - Config: Uses RapidAPI endpoint with headers for authentication and query parameters set dynamically.  
    - Connections: Outputs raw event data to "Filter Medium & High Impact News".  
    - Edge Cases: API downtime, rate limiting, invalid responses or JSON parsing errors.  

#### 2.3 Data Filtering

- **Overview:**  
  Filters the fetched data to include only events with medium or high impact.

- **Nodes Involved:**  
  - Filter Medium & High Impact News

- **Node Details:**  
  - **Filter Medium & High Impact News**  
    - Type: Code (JavaScript)  
    - Role: Parses the incoming event data and filters events by their impact rating.  
    - Config: Custom JS code checking event impact levels.  
    - Connections: Outputs filtered data to "Organizes Input".  
    - Edge Cases: Input data structure changes may break filtering logic; empty results if no events match criteria.  

#### 2.4 Data Organization

- **Overview:**  
  Organizes and formats filtered events into a structure suitable for AI processing.

- **Nodes Involved:**  
  - Organizes Input

- **Node Details:**  
  - **Organizes Input**  
    - Type: Code (JavaScript)  
    - Role: Structures filtered events into a formatted input, possibly summarizing or grouping data for AI consumption.  
    - Config: Custom JS preparing input text or JSON for AI agent.  
    - Connections: Outputs to "AI Agent".  
    - Edge Cases: Data formatting errors; empty inputs may lead to AI processing failures.  

#### 2.5 AI Enhancement

- **Overview:**  
  Uses Langchain with Google Gemini 2.0 chat model to enhance, format, and parse the economic calendar data into a user-friendly output.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model (Formats Output)  
  - Structured Output Parser

- **Node Details:**  
  - **Google Gemini Chat Model (Formats Output)**  
    - Type: Langchain Language Model Node  
    - Role: Provides the language model for generating enhanced text output.  
    - Config: Uses Google Gemini 2.0 with default or configured parameters.  
    - Connections: AI language model input for "AI Agent".  
    - Edge Cases: API quota limits, network errors, model service downtime.  

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Orchestrates AI processing with input data and language model, applies output parser to get structured results.  
    - Config: Uses "Structured Output Parser" as output parser node.  
    - Connections:  
      - Input: From "Organizes Input" and "Google Gemini Chat Model"  
      - Output: To "Update expression for text"  
    - Edge Cases: Parsing failures, model errors, timeouts.  

  - **Structured Output Parser**  
    - Type: Langchain Output Parser Node  
    - Role: Parses AI-generated text into structured data format for further automated processing.  
    - Config: Structured output parser for consistent AI response formatting.  
    - Connections: Output parser for "AI Agent".  
    - Edge Cases: Malformed AI output, parser misconfigurations.  

#### 2.6 Messaging

- **Overview:**  
  Prepares the final text message and sends the alert via Telegram.

- **Nodes Involved:**  
  - Update expression for text  
  - Send Upcoming IPO Calendar Updates via Telegram

- **Node Details:**  
  - **Update expression for text**  
    - Type: Code (JavaScript)  
    - Role: Updates or formats the AI-generated structured output into a final text message string.  
    - Config: Custom JS handling message text preparation for Telegram.  
    - Connections: Outputs to "Send Upcoming IPO Calendar Updates via Telegram".  
    - Edge Cases: Formatting errors leading to broken or incomplete messages.  

  - **Send Upcoming IPO Calendar Updates via Telegram**  
    - Type: Telegram Node  
    - Role: Sends the prepared message to a Telegram chat/channel.  
    - Config: Uses OAuth2 or Bot token credentials configured in n8n for Telegram API.  
    - Connections: Terminal node; no outputs.  
    - Edge Cases: Telegram API errors, credential expiration, message size limits.  

---

### 3. Summary Table

| Node Name                                | Node Type                              | Functional Role                       | Input Node(s)                       | Output Node(s)                              | Sticky Note                 |
|-----------------------------------------|--------------------------------------|------------------------------------|-----------------------------------|---------------------------------------------|-----------------------------|
| Schedule Every 7 Days                    | Schedule Trigger                     | Workflow trigger                   |                                   | Dynamically Sets the Date                    |                             |
| Dynamically Sets the Date                | Code (JavaScript)                    | Sets dynamic date parameters       | Schedule Every 7 Days              | Set API Key for RapidAPI & Dates             |                             |
| Set API Key for RapidAPI & Dates        | Set                                 | Sets API key and date parameters   | Dynamically Sets the Date          | Gets Upcoming News                           |                             |
| Gets Upcoming News                       | HTTP Request                        | Fetches economic calendar data     | Set API Key for RapidAPI & Dates  | Filter Medium & High Impact News             |                             |
| Filter Medium & High Impact News         | Code (JavaScript)                   | Filters events by impact level     | Gets Upcoming News                | Organizes Input                              |                             |
| Organizes Input                         | Code (JavaScript)                   | Formats data for AI processing     | Filter Medium & High Impact News  | AI Agent                                     |                             |
| Google Gemini Chat Model (Formats Output) | Langchain Language Model Node      | Provides AI language model         |                                   | AI Agent                                     |                             |
| Structured Output Parser                 | Langchain Output Parser             | Parses AI output into structured data |                                   | AI Agent (as output parser)                   |                             |
| AI Agent                                | Langchain Agent Node                | Orchestrates AI processing         | Organizes Input, Google Gemini Chat Model | Update expression for text                  |                             |
| Update expression for text               | Code (JavaScript)                   | Prepares final message text        | AI Agent                         | Send Upcoming IPO Calendar Updates via Telegram |                             |
| Send Upcoming IPO Calendar Updates via Telegram | Telegram Node                     | Sends alert message                | Update expression for text        |                                             |                             |
| Sticky Note (bc91168e...)                | Sticky Note                        | N/A                               |                                   |                                             |                             |
| Sticky Note1                            | Sticky Note                        | N/A                               |                                   |                                             |                             |
| Sticky Note2                            | Sticky Note                        | N/A                               |                                   |                                             |                             |
| Sticky Note3                            | Sticky Note                        | N/A                               |                                   |                                             |                             |
| Sticky Note4                            | Sticky Note                        | N/A                               |                                   |                                             |                             |
| Sticky Note5                            | Sticky Note                        | N/A                               |                                   |                                             |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `Schedule Every 7 Days`  
   - Type: Schedule Trigger  
   - Configure to run every 7 days (weekly).  

2. **Add a Code Node**  
   - Name: `Dynamically Sets the Date`  
   - Type: Code (JavaScript)  
   - Purpose: Calculate dynamic date ranges (e.g., today's date and a future date) to be used in API requests.  
   - Connect the output of `Schedule Every 7 Days` to this node.  
   - Sample logic: use JavaScript Date functions to generate ISO date strings.  

3. **Add a Set Node**  
   - Name: `Set API Key for RapidAPI & Dates`  
   - Type: Set  
   - Purpose: Store your RapidAPI key and set the date parameters for the HTTP request.  
   - Include parameters such as `apiKey`, `startDate`, `endDate` with values from previous node outputs.  
   - Connect from `Dynamically Sets the Date`.  
   - Ensure your RapidAPI key is secured in credentials or environment variables.  

4. **Add an HTTP Request Node**  
   - Name: `Gets Upcoming News`  
   - Type: HTTP Request  
   - Purpose: Call the external economic calendar API (through RapidAPI).  
   - Configure URL, headers (including `X-RapidAPI-Key`), and query parameters (`startDate`, `endDate`).  
   - Connect from `Set API Key for RapidAPI & Dates`.  

5. **Add a Code Node**  
   - Name: `Filter Medium & High Impact News`  
   - Type: Code (JavaScript)  
   - Purpose: Filter the fetched news events to retain only those with medium or high impact.  
   - Connect from `Gets Upcoming News`.  
   - Logic: Loop through the JSON response and filter based on event impact attribute.  

6. **Add a Code Node**  
   - Name: `Organizes Input`  
   - Type: Code (JavaScript)  
   - Purpose: Format and organize filtered news into a concise input for AI processing.  
   - Connect from `Filter Medium & High Impact News`.  
   - Prepare data as text or structured JSON as needed by AI agent.  

7. **Add a Langchain Language Model Node**  
   - Name: `Google Gemini Chat Model (Formats Output)`  
   - Type: Langchain Language Model Node  
   - Purpose: Use Google Gemini 2.0 to format and enhance the message.  
   - Configure with Google Gemini credentials or API access.  

8. **Add a Langchain Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Type: Langchain Output Parser Node  
   - Purpose: Parse AI output into structured data for reliable downstream use.  

9. **Add a Langchain Agent Node**  
   - Name: `AI Agent`  
   - Type: Langchain Agent Node  
   - Purpose: Coordinate AI processing using the language model and parser.  
   - Connect input from `Organizes Input` (main) and `Google Gemini Chat Model` (ai_languageModel).  
   - Set `Structured Output Parser` as the output parser.  
   - Connect output to next node.  

10. **Add a Code Node**  
    - Name: `Update expression for text`  
    - Type: Code (JavaScript)  
    - Purpose: Format the structured AI data into a final message string for Telegram.  
    - Connect from `AI Agent`.  

11. **Add a Telegram Node**  
    - Name: `Send Upcoming IPO Calendar Updates via Telegram`  
    - Type: Telegram  
    - Purpose: Send the message to a Telegram chat/channel.  
    - Configure Telegram credentials (bot token or OAuth2).  
    - Connect from `Update expression for text`.  

12. **Test the workflow**  
    - Trigger manually or wait for scheduled run.  
    - Verify messages received in Telegram with expected content.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow leverages Google Gemini 2.0 via Langchain for state-of-the-art AI text generation and structured parsing.                                   | AI enhancement with Langchain and Google Gemini           |
| RapidAPI key must be valid and have access to economic calendar APIs for reliable data fetching.                                                          | API key management and usage                               |
| Telegram node requires proper bot credentials and chat ID configured in n8n credentials for successful message delivery.                                | Telegram Bot API documentation: https://core.telegram.org/bots/api |
| Scheduling node depends on n8n instance uptime; consider external monitoring for critical alerts.                                                         | Scheduling best practices                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.