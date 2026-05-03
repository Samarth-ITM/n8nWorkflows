Automated Tech News  Report by ApiFlash, Gemini Vision & Sheets to Telegram

https://n8nworkflows.xyz/workflows/automated-tech-news--report-by-apiflash--gemini-vision---sheets-to-telegram-8291


# Automated Tech News  Report by ApiFlash, Gemini Vision & Sheets to Telegram

### 1. Workflow Overview

This workflow automates the creation and dissemination of a tech news report by integrating multiple services—ApiFlash for capturing web content images, Google Gemini AI for image analysis and text generation, Google Sheets for data storage and manipulation, and Telegram for messaging delivery. It is designed to be triggered on a schedule, collect and analyze tech news images, generate AI-based trend reports, store and aggregate data in sheets, and finally send photo and text updates to a Telegram channel or chat.

Logical blocks of the workflow:

- **1.1 Scheduled Trigger & Data Acquisition**: Starts the workflow on a schedule and fetches tech news images via an HTTP request.
- **1.2 Image Analysis & Messaging**: Processes the fetched images using Google Gemini AI, sends analyzed images as photo messages to Telegram.
- **1.3 Data Processing & Sheet Management**: Manipulates and stores data rows in Google Sheets, including filtering, editing, aggregating, and limiting data.
- **1.4 AI Trend Report Generation & Messaging**: Uses Google Gemini AI to generate a trend report from aggregated data and sends it as a text message on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Acquisition

- **Overview:**  
  Initiates the workflow on a defined schedule and performs an HTTP request to fetch images or data related to tech news.

- **Nodes Involved:**  
  - Schedule Trigger  
  - HTTP Request

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger  
    - Role: Initiates the workflow at scheduled intervals (e.g., daily, hourly).  
    - Configuration: Default schedule parameters likely set externally (not visible here).  
    - Inputs: None (trigger node)  
    - Outputs: HTTP Request  
    - Edge cases: Missed triggers if n8n instance is down; schedule misconfiguration.

  - **HTTP Request**  
    - Type: HTTP operation  
    - Role: Fetches images or data from an external API (likely ApiFlash capturing website screenshots).  
    - Configuration: URL and HTTP method not shown but expected to target tech news images or data endpoints.  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Analyze image, Send a photo message  
    - Edge cases: Network errors, API rate limits, invalid URLs, or unexpected response formats.

---

#### 1.2 Image Analysis & Messaging

- **Overview:**  
  Analyzes the fetched images using Google Gemini AI to interpret content, then sends the images as photo messages to Telegram.

- **Nodes Involved:**  
  - Analyze image  
  - Send a photo message  
  - Code (processing output)  
  - Split Out (splitting data)  
  - Append row in sheet

- **Node Details:**  

  - **Analyze image**  
    - Type: Google Gemini AI node for image analysis  
    - Role: Processes images from HTTP Request to extract insights or metadata.  
    - Configuration: Uses Google Gemini credentials; specifics not shown.  
    - Inputs: HTTP Request  
    - Outputs: Code  
    - Edge cases: AI service unavailability, invalid image data, authentication errors.

  - **Send a photo message**  
    - Type: Telegram node  
    - Role: Sends the fetched image as a photo message to a Telegram chat or channel.  
    - Configuration: Telegram credentials (OAuth2 likely), chat ID configured outside shown data.  
    - Inputs: HTTP Request (in parallel to Analyze image)  
    - Outputs: None  
    - Edge cases: Telegram API limits, invalid chat ID, network errors.

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Processes the AI analysis output, preparing data for further processing.  
    - Configuration: Custom JavaScript code (not shown) likely to format or extract relevant fields.  
    - Inputs: Analyze image  
    - Outputs: Split Out  
    - Edge cases: Code errors, unexpected data structure.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits the processed data into individual items for separate handling.  
    - Configuration: Default or custom splitting logic.  
    - Inputs: Code  
    - Outputs: Append row in sheet  
    - Edge cases: Empty data, splitting errors.

  - **Append row in sheet**  
    - Type: Google Sheets node  
    - Role: Appends each split data item as a new row in a Google Sheet.  
    - Configuration: Target spreadsheet and sheet specified externally; write permissions required.  
    - Inputs: Split Out  
    - Outputs: Aggregate  
    - Edge cases: API quota exceeded, permission errors, invalid data causing failures.

---

#### 1.3 Data Processing & Sheet Management

- **Overview:**  
  Reads, filters, edits, limits, and aggregates data from Google Sheets to prepare information for AI trend report generation.

- **Nodes Involved:**  
  - Aggregate  
  - Read entire Sheet  
  - Filter  
  - Edit Fields  
  - Limit  
  - Aggregate1

- **Node Details:**  

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates data after appending new rows; likely consolidates for next processing.  
    - Configuration: Aggregation criteria (sum, count, groupBy) not shown.  
    - Inputs: Append row in sheet  
    - Outputs: Read entire Sheet  
    - Edge cases: Empty data, aggregation errors.

  - **Read entire Sheet**  
    - Type: Google Sheets node  
    - Role: Reads all rows from the specified sheet for filtering and further processing.  
    - Configuration: Spreadsheet ID and sheet name set externally.  
    - Inputs: Aggregate  
    - Outputs: Filter  
    - Edge cases: Permission errors, large data causing timeouts.

  - **Filter**  
    - Type: Filter node  
    - Role: Applies conditions to data rows to narrow down relevant entries.  
    - Configuration: Filter expressions (e.g., by date, category) not shown.  
    - Inputs: Read entire Sheet  
    - Outputs: Edit Fields  
    - Edge cases: Incorrect filter logic causing no data or all data filtered out.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Modifies or adds fields to filtered data, preparing it for limiting.  
    - Configuration: Field values or expressions set to transform data.  
    - Inputs: Filter  
    - Outputs: Limit  
    - Edge cases: Expression errors or missing fields.

  - **Limit**  
    - Type: Limit node  
    - Role: Restricts the number of data items passing through (e.g., top N rows).  
    - Configuration: Limit count parameter (not shown).  
    - Inputs: Edit Fields  
    - Outputs: Aggregate1  
    - Edge cases: Limit value too high causing performance issues.

  - **Aggregate1**  
    - Type: Aggregate node  
    - Role: Final aggregation of limited and edited data to prepare for AI trend report.  
    - Configuration: Aggregation logic not detailed.  
    - Inputs: Limit  
    - Outputs: AI Trend Report  
    - Edge cases: Empty input, aggregation failures.

---

#### 1.4 AI Trend Report Generation & Messaging

- **Overview:**  
  Generates an AI-based trend report from aggregated data using Google Gemini AI and sends the report as a text message on Telegram.

- **Nodes Involved:**  
  - AI Trend Report  
  - Send a text message

- **Node Details:**  

  - **AI Trend Report**  
    - Type: Google Gemini AI node  
    - Role: Generates a textual trend report based on aggregated sheet data.  
    - Configuration: Uses Google Gemini credentials; prompt and parameters set externally.  
    - Inputs: Aggregate1  
    - Outputs: Send a text message  
    - Edge cases: AI processing errors, authentication issues.

  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends the generated AI trend report as a Telegram text message.  
    - Configuration: Telegram OAuth2 credentials configured; chat ID set outside JSON.  
    - Inputs: AI Trend Report  
    - Outputs: None  
    - Edge cases: Telegram API limits, invalid chat ID, network errors.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                    | Input Node(s)          | Output Node(s)             | Sticky Note |
|---------------------|----------------------------|----------------------------------|------------------------|----------------------------|-------------|
| Schedule Trigger     | scheduleTrigger            | Starts workflow on schedule      | -                      | HTTP Request               |             |
| HTTP Request        | httpRequest                | Fetches tech news images/data    | Schedule Trigger       | Analyze image, Send a photo message |             |
| Analyze image        | googleGemini (image analysis) | AI analyzes images              | HTTP Request           | Code                       |             |
| Send a photo message | telegram                   | Sends photo to Telegram          | HTTP Request           | -                          |             |
| Code                 | code                       | Processes AI output              | Analyze image          | Split Out                  |             |
| Split Out            | splitOut                   | Splits data items                | Code                   | Append row in sheet        |             |
| Append row in sheet  | googleSheets               | Adds data rows to sheet          | Split Out              | Aggregate                  |             |
| Aggregate            | aggregate                  | Aggregates appended data         | Append row in sheet    | Read entire Sheet          |             |
| Read entire Sheet    | googleSheets               | Reads all rows from sheet        | Aggregate              | Filter                     |             |
| Filter               | filter                     | Filters data rows                | Read entire Sheet      | Edit Fields                |             |
| Edit Fields          | set                        | Modifies/sets data fields        | Filter                 | Limit                      |             |
| Limit                | limit                      | Limits number of data items      | Edit Fields            | Aggregate1                 |             |
| Aggregate1           | aggregate                  | Final aggregation for report     | Limit                  | AI Trend Report            |             |
| AI Trend Report      | googleGemini (text generation) | Generates AI trend report       | Aggregate1             | Send a text message        |             |
| Send a text message  | telegram                   | Sends text message to Telegram   | AI Trend Report        | -                          |             |
| Sticky Note          | stickyNote                 | Comments / notes                 | -                      | -                          |             |
| Sticky Note1         | stickyNote                 | Comments / notes                 | -                      | -                          |             |
| Sticky Note2         | stickyNote                 | Comments / notes                 | -                      | -                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: scheduleTrigger  
   - Configure to run at desired intervals (e.g., daily at 8:00 AM).

2. **Add HTTP Request node:**  
   - Type: httpRequest  
   - Connect Schedule Trigger → HTTP Request  
   - Configure URL to fetch tech news images (e.g., ApiFlash screenshot API).  
   - Set HTTP Method (GET).  
   - Handle authentication if required.

3. **Add Analyze image node:**  
   - Type: googleGemini (image analysis)  
   - Connect HTTP Request → Analyze image  
   - Configure with Google Gemini credentials.  
   - Set parameters to analyze the image data received.

4. **Add Send a photo message node:**  
   - Type: telegram  
   - Connect HTTP Request → Send a photo message (parallel to Analyze image)  
   - Configure Telegram OAuth2 credentials.  
   - Set chat ID to target Telegram channel or user.  
   - Map photo content from HTTP Request data.

5. **Add Code node:**  
   - Type: code  
   - Connect Analyze image → Code  
   - Write JavaScript to process AI analysis output and prepare data for sheet insertion.

6. **Add Split Out node:**  
   - Type: splitOut  
   - Connect Code → Split Out  
   - Configure to split data array into individual items.

7. **Add Append row in sheet node:**  
   - Type: googleSheets (append row)  
   - Connect Split Out → Append row in sheet  
   - Configure with Google Sheets credentials.  
   - Set target Spreadsheet ID and Sheet name.  
   - Map fields to Google Sheets columns.

8. **Add Aggregate node:**  
   - Type: aggregate  
   - Connect Append row in sheet → Aggregate  
   - Configure aggregation (e.g., group by date or topic).

9. **Add Read entire Sheet node:**  
   - Type: googleSheets (read)  
   - Connect Aggregate → Read entire Sheet  
   - Configure with same Spreadsheet ID and Sheet name.  
   - Set to read all rows.

10. **Add Filter node:**  
    - Type: filter  
    - Connect Read entire Sheet → Filter  
    - Configure filter conditions (e.g., recent date, category).

11. **Add Edit Fields node:**  
    - Type: set  
    - Connect Filter → Edit Fields  
    - Configure to update or create fields needed for reporting.

12. **Add Limit node:**  
    - Type: limit  
    - Connect Edit Fields → Limit  
    - Set maximum number of rows to process (e.g., limit to 10).

13. **Add Aggregate1 node:**  
    - Type: aggregate  
    - Connect Limit → Aggregate1  
    - Configure final aggregation for AI input.

14. **Add AI Trend Report node:**  
    - Type: googleGemini (text generation)  
    - Connect Aggregate1 → AI Trend Report  
    - Configure with Google Gemini credentials.  
    - Set prompt to generate trend report based on aggregated data.

15. **Add Send a text message node:**  
    - Type: telegram  
    - Connect AI Trend Report → Send a text message  
    - Configure Telegram OAuth2 credentials and chat ID.  
    - Map AI report text to message content.

16. **Verify all connections and credentials:**  
    - Ensure Google Gemini credentials are active and authorized.  
    - Validate Google Sheets API access and permissions.  
    - Confirm Telegram bot or user OAuth2 credentials with messaging rights.  
    - Test each node individually for expected outputs.

17. **Optional:** Add Sticky Notes for documentation inside n8n as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                     |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Workflow leverages Google Gemini AI nodes for both image analysis and text generation, requiring proper Google API credentials. | Google Gemini API documentation                     |
| Telegram nodes require OAuth2 credentials and correct chat IDs for messaging. Ensure bot permissions to send messages. | Telegram Bot API reference                          |
| ApiFlash or similar services can provide screenshots of web pages for image input to AI analysis.             | https://apiflash.com/                              |
| Google Sheets nodes require OAuth2 credentials with read/write access to the target spreadsheet.              | Google Sheets API documentation                      |
| Ensure n8n environment has stable internet and API quota management to avoid timeouts or rate limiting issues.| n8n best practices for API integrations             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.