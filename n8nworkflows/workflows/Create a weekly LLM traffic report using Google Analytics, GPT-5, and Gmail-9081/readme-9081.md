Create a weekly LLM traffic report using Google Analytics, GPT-5, and Gmail

https://n8nworkflows.xyz/workflows/create-a-weekly-llm-traffic-report-using-google-analytics--gpt-5--and-gmail-9081


# Create a weekly LLM traffic report using Google Analytics, GPT-5, and Gmail

### 1. Workflow Overview

This workflow automates the creation and emailing of a weekly traffic report focused on visitors originating from large language model (LLM) related sources (e.g., ChatGPT, Gemini). It integrates Google Analytics to fetch traffic data, uses an advanced AI model (GPT-5) to generate a formatted report, and sends the report via Gmail.

Logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow every week on Monday.
- **1.2 Data Retrieval from Google Analytics**: Fetches sessions by source and medium for the previous week.
- **1.3 Traffic Filtering**: Filters traffic sources to include only known LLM-related referral domains.
- **1.4 Data Aggregation**: Combines filtered session data into a single dataset.
- **1.5 AI Report Generation**: Uses GPT-5 via OpenAI integration to create a structured HTML traffic report.
- **1.6 Email Dispatch**: Sends the generated report by email with contextual subject lines reflecting the reporting period.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview**: This block triggers the entire workflow automatically every week, precisely on Monday.
- **Nodes Involved**:  
  - Every week...
- **Node Details**:  
  - **Every week...**
    - Type: Schedule Trigger  
    - Role: Initiates workflow on a weekly interval.  
    - Configuration: Set to trigger every Monday (weekly interval, triggerAtDay: 1).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Get sessions by source & medium".  
    - Edge cases: Workflow will not run if n8n instance is down at trigger time; no retries built-in here.  

#### 2.2 Data Retrieval from Google Analytics

- **Overview**: Retrieves session data broken down by source and medium from Google Analytics for the previous week.
- **Nodes Involved**:  
  - Get sessions by source & medium
- **Node Details**:  
  - **Get sessions by source & medium**
    - Type: Google Analytics (GA4) node  
    - Role: Fetches traffic metrics from GA4 API.  
    - Configuration:  
      - Metrics: Sessions (all returned).  
      - Dimensions: sourceMedium (source / medium combination).  
      - Property ID: User must configure with their GA4 Property ID.  
    - Credentials: Google Analytics OAuth2 (configured with valid user credentials).  
    - Inputs: Triggered by the schedule trigger node.  
    - Outputs: Session data passed to filtering node.  
    - Edge cases:  
      - Invalid or missing GA property ID causes API errors.  
      - OAuth token expiration or revocation can cause auth failures.  
      - API quota limits could cause throttling.  
    - Notes: Sticky note advises where to find the GA property ID.

#### 2.3 Traffic Filtering

- **Overview**: Filters the fetched traffic data to include only sources related to LLMs using a regex pattern.
- **Nodes Involved**:  
  - Filter known referral domains from AI companies
- **Node Details**:  
  - **Filter known referral domains from AI companies**
    - Type: Code (JavaScript) node  
    - Role: Filters session items by matching source/medium against a regex of LLM-related domains.  
    - Configuration:  
      - Regex includes patterns for domains like openai, copilot, chatgpt, gemini, gpt, neeva, bard, writesonic, perplexity, and others.  
      - The code filters the items array by checking if `sourceMedium` matches the regex (case-insensitive).  
    - Inputs: Receives all session items from the Google Analytics node.  
    - Outputs: Filtered list only including LLM traffic sources.  
    - Edge cases:  
      - If sourceMedium property is missing or malformed, item is excluded.  
      - Regex must be updated manually to add/remove domains as needed.  
      - No fallback if no matches found; downstream nodes must handle empty input gracefully.

#### 2.4 Data Aggregation

- **Overview**: Aggregates all filtered session data items into a single combined dataset to simplify processing for the AI.
- **Nodes Involved**:  
  - Combine items
- **Node Details**:  
  - **Combine items**
    - Type: Aggregate node  
    - Role: Merges all filtered items into one consolidated output.  
    - Configuration: Uses "aggregateAllItemData" option to combine all JSON data arrays.  
    - Inputs: Receives filtered items from the previous Code node.  
    - Outputs: Single combined dataset passed to the AI node.  
    - Edge cases:  
      - If input is empty, output will be empty; AI node must handle empty data gracefully.

#### 2.5 AI Report Generation

- **Overview**: Sends the aggregated traffic data to GPT-5 to generate a structured weekly traffic report formatted in HTML for email.
- **Nodes Involved**:  
  - Create traffic report
- **Node Details**:  
  - **Create traffic report**
    - Type: OpenAI (Langchain) node  
    - Role: Uses GPT-5 to create a weekly report from the combined traffic data.  
    - Configuration:  
      - Model: GPT-5 selected from dropdown.  
      - Messages:  
        - User prompt includes the JSON stringified traffic data (`{{ $json.data.toJsonString() }}`).  
        - System prompt instructs the AI to generate a report with a specific structure: a table of top sources with columns for rank, source/medium, and sessions.  
        - Instructions restrict output to email body HTML only, no extra commentary.  
      - Options: Default.  
    - Credentials: Uses OpenAI API Key credential.  
    - Inputs: Receives combined session data.  
    - Outputs: Passes generated message content to the email node.  
    - Edge cases:  
      - API rate limits or authentication failure could cause errors.  
      - AI prompt may not always format perfectly; testing and prompt tuning recommended.  
      - Empty or malformed data input may result in poor or empty reports.

#### 2.6 Email Dispatch

- **Overview**: Sends the generated traffic report as an email with a subject line reflecting the week number and date range.
- **Nodes Involved**:  
  - Send report
- **Node Details**:  
  - **Send report**
    - Type: Gmail node  
    - Role: Sends the generated HTML report via Gmail.  
    - Configuration:  
      - Send To: Configured email address (placeholder "youremail@example.com" must be replaced).  
      - Message: Uses the AI-generated content (`{{$json.message.content}}`).  
      - Subject: Dynamically created with last week's week number and date range using n8n date expressions.  
      - Options: Attribution appended disabled for cleaner emails.  
    - Credentials: Gmail OAuth2 (configured with valid Gmail account credentials).  
    - Inputs: Receives report content from the AI node.  
    - Outputs: None (terminal node).  
    - Edge cases:  
      - Gmail API quota or credential failures may block email sending.  
      - Invalid recipient email or formatting issues may cause delivery failure.  
      - Large email body size may be truncated or rejected.

---

### 3. Summary Table

| Node Name                              | Node Type                            | Functional Role                           | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                                             |
|--------------------------------------|------------------------------------|-----------------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Every week...                        | Schedule Trigger                   | Triggers workflow weekly on Monday      | None                        | Get sessions by source & medium | Run workflow every week                                                                                                                 |
| Get sessions by source & medium      | Google Analytics (GA4)             | Retrieves weekly session data by source/medium | Every week...               | Filter known referral domains   | Get last weeks sessions by source / medium. Add your Google Analytics property ID to this node.                                          |
| Filter known referral domains from AI companies | Code (JavaScript)                  | Filters traffic to known LLM referral domains | Get sessions by source & medium | Combine items                  | Filter for LLM traffic. Regex matches domains like ChatGPT, Gemini, Perplexity, etc. Add your own domains as needed.                     |
| Combine items                       | Aggregate                         | Combines filtered items into one dataset | Filter known referral domains | Create traffic report          | Combine all items into one list for the LLM                                                                                            |
| Create traffic report                | OpenAI (Langchain)                 | Generates a weekly HTML traffic report using GPT-5 | Combine items               | Send report                    | Generate report with AI. Produces email-friendly HTML with traffic table                                                                |
| Send report                        | Gmail                            | Sends the generated report via email    | Create traffic report         | None                          | Send report via email. Subject includes last week's week number and dates                                                               |
| Sticky Note                        | Sticky Note                      | Informational                          | None                        | None                          | Overview: Get a weekly report on website traffic driven by LLMs like ChatGPT, Perplexity, and Gemini. Setup instructions included.        |
| Sticky Note1                       | Sticky Note                      | Informational                          | None                        | None                          | Get sessions by source / medium. Locate Google Analytics Property ID in admin panel                                                      |
| Sticky Note2                       | Sticky Note                      | Informational                          | None                        | None                          | Filter for LLM traffic, regex-based                                                                                                    |
| Sticky Note3                       | Sticky Note                      | Informational                          | None                        | None                          | Combine items into one list for AI input                                                                                                |
| Sticky Note4                       | Sticky Note                      | Informational                          | None                        | None                          | Generate report with AI in email-friendly HTML                                                                                          |
| Sticky Note5                       | Sticky Note                      | Informational                          | None                        | None                          | Send report by email with dynamic subject                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday (interval: weeks, triggerAtDay: 1).

2. **Add Google Analytics GA4 node**  
   - Type: Google Analytics  
   - Credentials: Connect Google Analytics OAuth2 credentials with access to your GA4 property.  
   - Parameters:  
     - Property ID: Enter your GA4 Property ID (found under Admin > Property Settings in GA).  
     - Metrics: Select "Sessions" (return all).  
     - Dimensions: Select "sourceMedium".  
   - Connect output of Schedule Trigger to this node.

3. **Add Code node to filter LLM traffic**  
   - Type: Code (JavaScript)  
   - Paste this code to filter traffic sources by regex matching known LLM referral domains:
     ```javascript
     const regex = /^.openai.*|.*copilot.*|.*chatgpt.*|.*gemini.*|.*gpt.*|.*neeva.*|.*writesonic.*|.*nimble.*|.*outrider.*|.*perplexity.*|.*google.*bard.*|.*bard.*google.*|.*bard.*|.*edgeservices.*|.*astastic.*|.*copy.ai.*|.*bnngpt.*|.*gemini.*google.*$/i;
     return items.filter(item => {
       const value = item.json.sourceMedium || "";
       return regex.test(value);
     });
     ```
   - Connect output of Google Analytics node to this Code node.

4. **Add Aggregate node to combine filtered items**  
   - Type: Aggregate  
   - Set option to "aggregateAllItemData" to combine all JSON data into one output.  
   - Connect output of Code node to this Aggregate node.

5. **Add OpenAI (Langchain) node to generate the report**  
   - Type: OpenAI (Langchain)  
   - Credentials: Connect your OpenAI API key.  
   - Parameters:  
     - Model: Select "gpt-5".  
     - Messages:  
       - User message:  
         ```
         Please create a report of the following traffic

         {{ $json.data.toJsonString() }}
         ```
       - System message:  
         ```
         You are a virtual assistant. Your job is to take the list of traffic numbers provided and generate a weekly report that will be sent by email. Use the following structure:

         <structure>
         ## Top sources
         Show a table with top sources of traffic. The columns should be | # | Source / Medium | Sessions |
         </structure>

         Adhere to the following rules:
         - Only return the body text of the email.
         - Return the body text without any additional commentary or remarks
         - The report should be formatted to comply with HTML used in emails.
         ```
   - Connect output of Aggregate node to this OpenAI node.

6. **Add Gmail node to send the report**  
   - Type: Gmail  
   - Credentials: Connect Gmail OAuth2 credentials.  
   - Parameters:  
     - Send To: Enter your target recipient email address.  
     - Subject: Use expression to set subject dynamically:  
       ```
       LLM traffic report for week {{$now.minus(1,'week').format('W')}} - {{$now.minus(1,'week').format('dd MMMM yyyy')}} to {{$now.format('dd MMMM yyyy')}}
       ```
     - Message: Use expression: `{{$json.message.content}}` to get the email body from the AI node.  
     - Options: Disable append attribution.  
   - Connect output of OpenAI node to this Gmail node.

7. **Test the workflow end to end**  
   - Ensure all credentials are valid and API limits are respected.  
   - Adjust regex in Code node if needed to capture additional or fewer domains.  
   - Confirm that the Google Analytics Property ID is correct and data is accessible.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow provides a weekly snapshot of website traffic from LLM-related sources to inform content and marketing decisions. | Overview sticky note in workflow.                                                                             |
| Google Analytics Property ID is found under Admin > Property Settings in GA4.                                  | Sticky Note1 guidance.                                                                                        |
| Regex in filtering node can be customized to add or remove referral domains as LLM ecosystem evolves.         | Sticky Note2 instruction.                                                                                     |
| AI report is generated with GPT-5 using Langchain integration, outputting HTML formatted for email clients.    | Sticky Note4 description.                                                                                     |
| Email subject dynamically reflects the previous week’s number and date range using n8n date functions.         | Sticky Note5 description.                                                                                     |
| Gmail OAuth2 credentials require prior setup via Google Cloud Console with appropriate scopes enabled.         | General n8n Gmail node documentation.                                                                         |
| OpenAI API key must have access to GPT-5 model (or equivalent) and be configured in n8n credentials.            | General n8n OpenAI integration documentation.                                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow designed for integration and automation purposes. It complies strictly with content policies and contains no illegal or offensive elements. All handled data is legal and publicly accessible.