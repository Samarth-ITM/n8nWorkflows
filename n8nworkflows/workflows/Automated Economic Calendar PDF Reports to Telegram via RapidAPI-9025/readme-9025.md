Automated Economic Calendar PDF Reports to Telegram via RapidAPI

https://n8nworkflows.xyz/workflows/automated-economic-calendar-pdf-reports-to-telegram-via-rapidapi-9025


# Automated Economic Calendar PDF Reports to Telegram via RapidAPI

### 1. Workflow Overview

This workflow automates the generation and delivery of weekly economic calendar reports in PDF format to a Telegram channel or user. It leverages RapidAPI to retrieve upcoming economic news events, processes and filters these events by impact level, dynamically generates a customized PDF report using an external API, and finally sends the report through Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Date Setup:** Automatically triggers the process every 7 days and dynamically sets the required date parameters for the API query.
- **1.2 API Request & Data Retrieval:** Sets required API credentials and parameters, then fetches upcoming economic news/events data from RapidAPI.
- **1.3 Data Filtering & Preparation:** Filters the retrieved news for medium and high impact events and consolidates the data into a single array for processing.
- **1.4 Report Generation:** Organizes the filtered data into a template format, edits/updates a report template via an API call, and downloads the resulting PDF report.
- **1.5 Report Delivery:** Sends the downloaded PDF report to Telegram via a Telegram node.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Date Setup

- **Overview:** This block initiates the workflow execution on a weekly basis and sets dynamic date parameters for the economic calendar API requests.
- **Nodes Involved:**  
  - Schedule Every 7 Days  
  - Dynamically Sets the Date  
  - Set API Key for RapidAPI & Dates  

- **Node Details:**

  - **Schedule Every 7 Days**  
    - *Type:* Schedule Trigger  
    - *Role:* Triggers workflow execution every 7 days automatically.  
    - *Configuration:* Default schedule set to trigger weekly.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to "Dynamically Sets the Date".  
    - *Edge Cases:* Workflow will not trigger if n8n instance is offline or scheduler is disabled.

  - **Dynamically Sets the Date**  
    - *Type:* Code (JavaScript)  
    - *Role:* Computes date ranges (start and end dates) dynamically based on current date or schedule.  
    - *Configuration:* Custom JavaScript code generating dates in required format for API queries.  
    - *Key Expressions:* Uses JavaScript Date objects and formatting.  
    - *Inputs:* Trigger from Schedule node.  
    - *Outputs:* Connects to "Set API Key for RapidAPI & Dates".  
    - *Edge Cases:* Potential errors if date calculations fail or produce invalid formats.

  - **Set API Key for RapidAPI & Dates**  
    - *Type:* Set  
    - *Role:* Sets the API key for RapidAPI and constructs request parameters (including the dates) for the economic calendar API call.  
    - *Configuration:* Sets static or credential-based API key and merges date parameters from previous node.  
    - *Inputs:* Receives date data from "Dynamically Sets the Date".  
    - *Outputs:* Connects to "Gets Upcoming News".  
    - *Edge Cases:* Missing or invalid API key will cause authentication failures downstream.

---

#### 1.2 API Request & Data Retrieval

- **Overview:** Makes the HTTP request to RapidAPI to fetch upcoming economic news/events using the prepared parameters and key.
- **Nodes Involved:**  
  - Gets Upcoming News  

- **Node Details:**

  - **Gets Upcoming News**  
    - *Type:* HTTP Request  
    - *Role:* Calls the RapidAPI endpoint to get upcoming economic news events.  
    - *Configuration:*  
      - Method: GET (or as required by API)  
      - URL: RapidAPI endpoint URL (specific to economic calendar)  
      - Headers: Include API Key and required headers  
      - Query Parameters: Include dynamic dates and other filters set earlier  
    - *Inputs:* Receives parameters from "Set API Key for RapidAPI & Dates".  
    - *Outputs:* Connects to "Filter Medium & High Impact News".  
    - *Edge Cases:* Network errors, API rate limiting, invalid API keys, data format changes.

---

#### 1.3 Data Filtering & Preparation

- **Overview:** Filters the news items to only include those with medium or high impact and consolidates the filtered items into a single array for template processing.
- **Nodes Involved:**  
  - Filter Medium & High Impact News  
  - Convert All Items Into One Array  

- **Node Details:**

  - **Filter Medium & High Impact News**  
    - *Type:* Code (JavaScript)  
    - *Role:* Filters the array of news items, keeping only those with medium or high impact.  
    - *Configuration:* Custom JavaScript filtering logic based on impact attribute.  
    - *Inputs:* Receives news items from "Gets Upcoming News".  
    - *Outputs:* Connects to "Convert All Items Into One Array".  
    - *Edge Cases:* Data structure changes, missing impact field, empty results.

  - **Convert All Items Into One Array**  
    - *Type:* Code (JavaScript)  
    - *Role:* Flattens or consolidates multiple news items into a single array object, suitable for API template consumption.  
    - *Configuration:* JavaScript code that merges or restructures data arrays.  
    - *Inputs:* Filtered data from previous node.  
    - *Outputs:* Connects to "Organize Data for API Template".  
    - *Edge Cases:* Data format inconsistencies, empty input arrays.

---

#### 1.4 Report Generation

- **Overview:** Formats the filtered news data into the report template, updates the template via an external API, and downloads the resulting PDF report.
- **Nodes Involved:**  
  - Organize Data for API Template  
  - Edit & Update Template  
  - Download PDF Report  

- **Node Details:**

  - **Organize Data for API Template**  
    - *Type:* Code (JavaScript)  
    - *Role:* Transforms consolidated data into the exact JSON structure or payload required by the external report generation API.  
    - *Configuration:* Custom JavaScript to format and fill template placeholders.  
    - *Inputs:* Receives consolidated array from "Convert All Items Into One Array".  
    - *Outputs:* Connects to "Edit & Update Template".  
    - *Edge Cases:* Incorrect data mapping, missing required fields.

  - **Edit & Update Template**  
    - *Type:* HTTP Request  
    - *Role:* Sends the formatted data to an external service (likely via RapidAPI) to update the report template content.  
    - *Configuration:* POST or PUT method with JSON payload containing report data.  
    - *Inputs:* Receives organized data from previous node.  
    - *Outputs:* Connects to "Download PDF Report".  
    - *Edge Cases:* API errors, authentication failures, payload schema mismatch.

  - **Download PDF Report**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the generated PDF report from the report generation service.  
    - *Configuration:* GET request to a provided URL or endpoint returning the PDF file.  
    - *Inputs:* Receives response from "Edit & Update Template" (likely containing a download URL).  
    - *Outputs:* Connects to "Send Economic Calendar Events PDF to Telegram".  
    - *Edge Cases:* Download failures, timeouts, invalid URLs.

---

#### 1.5 Report Delivery

- **Overview:** Sends the downloaded PDF report file to a Telegram chat or channel.
- **Nodes Involved:**  
  - Send Economic Calendar Events PDF to Telegram  

- **Node Details:**

  - **Send Economic Calendar Events PDF to Telegram**  
    - *Type:* Telegram  
    - *Role:* Sends the PDF report as a file attachment to a configured Telegram chat or channel.  
    - *Configuration:*  
      - Authentication via Telegram Bot API token (OAuth2 or token credentials)  
      - Chat ID or username to send the file to  
      - File input: PDF binary data from previous node  
    - *Inputs:* Receives the PDF binary data from "Download PDF Report".  
    - *Outputs:* None (end of workflow).  
    - *Edge Cases:* Bot permissions, chat ID errors, file size limits, Telegram API rate limits.

---

### 3. Summary Table

| Node Name                            | Node Type         | Functional Role                              | Input Node(s)                      | Output Node(s)                            | Sticky Note                              |
|------------------------------------|-------------------|----------------------------------------------|----------------------------------|------------------------------------------|-----------------------------------------|
| Schedule Every 7 Days               | Schedule Trigger  | Triggers workflow weekly                      | None                             | Dynamically Sets the Date                 |                                         |
| Dynamically Sets the Date           | Code              | Calculates dynamic date parameters            | Schedule Every 7 Days             | Set API Key for RapidAPI & Dates          |                                         |
| Set API Key for RapidAPI & Dates   | Set               | Sets API authentication and request parameters| Dynamically Sets the Date         | Gets Upcoming News                        |                                         |
| Gets Upcoming News                 | HTTP Request      | Fetches economic news from RapidAPI           | Set API Key for RapidAPI & Dates | Filter Medium & High Impact News          |                                         |
| Filter Medium & High Impact News  | Code              | Filters news by impact level                    | Gets Upcoming News               | Convert All Items Into One Array           |                                         |
| Convert All Items Into One Array   | Code              | Consolidates filtered news into one array      | Filter Medium & High Impact News | Organize Data for API Template             |                                         |
| Organize Data for API Template     | Code              | Formats data for report template API           | Convert All Items Into One Array | Edit & Update Template                    |                                         |
| Edit & Update Template            | HTTP Request      | Updates report template with data              | Organize Data for API Template   | Download PDF Report                       |                                         |
| Download PDF Report                | HTTP Request      | Downloads generated PDF report                  | Edit & Update Template           | Send Economic Calendar Events PDF to Telegram |                                         |
| Send Economic Calendar Events PDF to Telegram | Telegram          | Sends the PDF report to Telegram                | Download PDF Report              | None                                     |                                         |
| Sticky Note2                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note6                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note7                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note                        | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note1                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note3                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note4                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note5                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |
| Sticky Note8                       | Sticky Note       | (No content)                                   | None                             | None                                     |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: `Schedule Every 7 Days`
   - Type: Schedule Trigger
   - Configuration: Set to trigger every 7 days (weekly).
   
2. **Create a Code node:**
   - Name: `Dynamically Sets the Date`
   - Type: Code (JavaScript)
   - Configuration: Write JavaScript to calculate the date range (start and end dates) for the upcoming week or desired period. Format dates as required by the API.
   - Connect input from `Schedule Every 7 Days`.

3. **Create a Set node:**
   - Name: `Set API Key for RapidAPI & Dates`
   - Type: Set
   - Configuration:  
     - Add field(s) for API Key (can be set statically or via credentials)  
     - Add date parameters obtained from the previous code node  
     - Prepare any other request headers or query parameters needed for RapidAPI.
   - Connect input from `Dynamically Sets the Date`.

4. **Create an HTTP Request node:**
   - Name: `Gets Upcoming News`
   - Type: HTTP Request
   - Configuration:  
     - Method: GET or as specified by RapidAPI for the economic calendar endpoint  
     - URL: RapidAPI endpoint URL for economic news/events  
     - Headers: Include API Key header from `Set API Key for RapidAPI & Dates`  
     - Query Parameters: Include the dynamic dates and any other filters  
   - Connect input from `Set API Key for RapidAPI & Dates`.

5. **Create a Code node:**
   - Name: `Filter Medium & High Impact News`
   - Type: Code (JavaScript)
   - Configuration: Write JavaScript to filter the incoming array of news items. Keep only those with impact labeled as "medium" or "high".
   - Connect input from `Gets Upcoming News`.

6. **Create a Code node:**
   - Name: `Convert All Items Into One Array`
   - Type: Code (JavaScript)
   - Configuration: Consolidate filtered news items into a single array or object matching the expected input format for the report template API.
   - Connect input from `Filter Medium & High Impact News`.

7. **Create a Code node:**
   - Name: `Organize Data for API Template`
   - Type: Code (JavaScript)
   - Configuration: Format and organize the array data into the exact JSON structure or template payload required by the report generation API.
   - Connect input from `Convert All Items Into One Array`.

8. **Create an HTTP Request node:**
   - Name: `Edit & Update Template`
   - Type: HTTP Request
   - Configuration:  
     - Method: POST or PUT as required  
     - URL: Endpoint of the report generation API (RapidAPI or other)  
     - Headers: Include authentication if needed  
     - Body: JSON containing the organized data from the previous node  
   - Connect input from `Organize Data for API Template`.

9. **Create an HTTP Request node:**
   - Name: `Download PDF Report`
   - Type: HTTP Request
   - Configuration:  
     - Method: GET  
     - URL: Use the URL or link obtained from the previous node's response to download the PDF file  
     - Response Format: Set to download binary data (PDF)  
   - Connect input from `Edit & Update Template`.

10. **Create a Telegram node:**
    - Name: `Send Economic Calendar Events PDF to Telegram`
    - Type: Telegram
    - Configuration:  
      - Credentials: Set up with Telegram Bot API token (OAuth2 or token-based)  
      - Chat ID or username: Set recipient chat/channel  
      - File: Use binary data from `Download PDF Report` as the file to send  
    - Connect input from `Download PDF Report`.

11. **Optionally add Sticky Notes for documentation:**
    - Use sticky notes at strategic positions to document purpose or instructions for each block or node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow automates weekly economic calendar report delivery via Telegram using RapidAPI and PDF generation. | Project purpose summary                                   |
| Ensure RapidAPI key is valid and has quota for economic calendar endpoints and PDF generation APIs.       | API Key management                                        |
| Telegram Bot must have permissions to send files to target chat/channel.                                   | Telegram Bot API requirements                             |
| For dynamic date calculations, verify time zones and date formats expected by the API.                     | Date handling best practice                               |
| PDF generation depends on the external API's template structure and payload format.                        | Template API documentation (external to this workflow)   |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.