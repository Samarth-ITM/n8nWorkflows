Create & Send Automated GA4 KPI Reports with AI Insights & Gmail

https://n8nworkflows.xyz/workflows/create---send-automated-ga4-kpi-reports-with-ai-insights---gmail-8463


# Create & Send Automated GA4 KPI Reports with AI Insights & Gmail

### 1. Workflow Overview

This workflow automates the creation and emailing of detailed GA4 (Google Analytics 4) KPI reports enriched with AI-generated insights and recommendations. The report includes comparative analysis between two defined periods, highlighting key traffic, engagement, and conversion metrics. It targets marketing, growth, and analytics professionals who require data-driven, insightful monthly summaries delivered as polished HTML email reports.

The workflow’s logic is organized into these main functional blocks:

- **1.1 Input Reception:** Collects client-specific GA4 details, date ranges, and email recipient via a form.
- **1.2 Data Retrieval:** Fetches current and previous period GA4 metrics and event counts using Google Analytics Data API.
- **1.3 Data Formatting:** Normalizes date formats and prepares raw data for AI analysis.
- **1.4 AI Reporting:** An AI Agent calculates percentage changes, generates textual summaries, recommendations, and composes a full valid HTML report.
- **1.5 Email Dispatch:** Sends the generated HTML report via Gmail with a contextual subject line.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block collects all necessary inputs from the user via a form trigger, including GA4 Account ID, key event name, date ranges (current and previous), client name, and recipient email.

- **Nodes Involved:**
  - `Get Client`

- **Node Details:**

  - **Get Client**
    - Type: Form Trigger (n8n-nodes-base.formTrigger)
    - Role: Entry point for user data; triggers workflow on form submission.
    - Configuration:
      - Custom HTML explanation embedded in form introduction.
      - Fields collected:
        - Account ID (required; GA4 Property ID)
        - Key Event (required; e.g., "form_submit")
        - Report from start date (required)
        - Report to this end date (required)
        - Previous start date (required)
        - Previous end date (required)
        - Client Name (required)
        - Send to email (required)
      - Webhook ID defined for form submissions.
    - Inputs: None (trigger node).
    - Outputs: Emits form data as JSON.
    - Edge Cases/Potential Failures:
      - Missing or invalid form fields can cause downstream failures.
      - Date fields must be valid dates.
      - Malformed email addresses could prevent successful email sending.
    - Version-specific: Uses version 2.2, standard form trigger.

---

#### 2.2 Data Retrieval

- **Overview:** This block calls Google Analytics Data API to retrieve metrics and event counts for the current and previous periods for comparison.

- **Nodes Involved:**
  - `Overall Metrics This Period`
  - `Overall Metrics Previous Period`
  - `Form Submits This Period`
  - `Form Submits Previous Period`

- **Node Details:**

  - **Overall Metrics This Period**
    - Type: HTTP Request
    - Role: Fetches GA4 aggregated metrics for the current report period.
    - Configuration:
      - POST request to Google Analytics Data API endpoint: `https://analyticsdata.googleapis.com/v1beta/properties/{{ Account ID }}:runReport`
      - Body requests metrics: sessions, engagedSessions, averageSessionDuration, bounceRate, screenPageViewsPerSession, eventsPerSession, newUsers, activeUsers, sessionKeyEventRate (for the key event).
      - Date range set to "Report from start date" to "Report to this end date" from form input.
      - Uses Google Analytics OAuth2 credential.
    - Inputs: Receives form data from `Get Client`.
    - Outputs: JSON with GA4 metrics for the current period.
    - Edge Cases:
      - API authentication failures.
      - Invalid or missing Account ID.
      - Rate limits or API timeouts.
    - Version: 4.2.

  - **Overall Metrics Previous Period**
    - Same as above but for previous period defined by "Previous start date" to "Previous end date".
    - Inputs: From `Overall Metrics This Period`.
    - Outputs: JSON with GA4 metrics for the previous period.
    - Same potential failure points as above.

  - **Form Submits This Period**
    - Type: HTTP Request
    - Role: Fetches event count (conversion count) for the key event in the current period.
    - Configuration:
      - POST to same GA4 Data API endpoint.
      - Metrics: eventCount.
      - Filters by eventName matching the key event.
      - Date range: current period.
      - Uses Google Analytics OAuth2 credential.
    - Inputs: From `Overall Metrics Previous Period`.
    - Outputs: JSON with event count data.
    - Edge Cases:
      - Key event name must be valid and exact.
      - API auth and rate limits.
    - Version: 4.2.

  - **Form Submits Previous Period**
    - Same as above but for previous period date range.
    - Inputs: From `Form Submits This Period`.
    - Outputs: JSON with previous period event count.
    - Edge Cases: Same as above.

---

#### 2.3 Data Formatting

- **Overview:** Prepares and normalizes form data, specifically reformats date strings by removing hyphens to comply with API requirements or internal formatting.

- **Nodes Involved:**
  - `Code`

- **Node Details:**

  - **Code**
    - Type: Code node (JavaScript)
    - Role: Removes hyphens from date strings in form data.
    - Configuration:
      - Reads form data JSON.
      - Iterates over date fields: "Report from start date", "Report to this end date", "Previous start date", "Previous end date".
      - Removes hyphens from these date strings.
      - Outputs one item with updated JSON.
    - Inputs: From `Form Submits Previous Period`.
    - Outputs: Reformatted form data ready for AI processing.
    - Edge Cases:
      - Date fields missing or in unexpected format may not be processed correctly.
    - Version: 2.

---

#### 2.4 AI Reporting

- **Overview:** This critical block uses an AI Agent to analyze all retrieved GA4 data, calculate percentage changes, generate textual summaries, recommendations, and produce a complete, valid HTML report with color-coded metrics reflecting performance changes.

- **Nodes Involved:**
  - `AI Agent`
  - `OpenAI Chat Model`
  - `Calculator`
  - `Think`

- **Node Details:**

  - **AI Agent**
    - Type: LangChain Agent node
    - Role: Central AI processing unit that consumes metrics and generates the full HTML report.
    - Configuration:
      - Inputs dynamic JSON data from various previous nodes (form data, current/previous metrics, current/previous conversions).
      - Embedded detailed prompt guiding AI to:
        - Calculate percent changes with precise logic for positive/negative coloring.
        - Generate two summary paragraphs focusing separately on traffic and engagement/conversion.
        - Create recommendations in grouped sections (engagement, traffic, conversion).
        - Output only the final valid HTML report (no extra commentary or code fences).
        - Validate HTML correctness.
      - Uses "Think" and "Calculator" tools internally to ensure calculations and reasoning accuracy.
      - Limits iterations to max 50 for robustness.
      - Uses OpenAI GPT-4.1-mini model via the linked OpenAI Chat Model node.
    - Inputs:
      - Receives reformatted form data from `Code`.
      - Uses AI tools `Calculator` and `Think` for calculations and data validation.
      - Uses `OpenAI Chat Model` as the language model backend.
    - Outputs:
      - Final HTML report string.
    - Edge Cases:
      - AI hallucination risks mitigated by strict prompt instructions.
      - Calculation errors if input data is malformed.
      - API rate limits or token limits in OpenAI.
    - Version: 1.8.

  - **OpenAI Chat Model**
    - Type: LangChain OpenAI Chat Model node
    - Role: Provides GPT-4.1-mini model for the AI Agent.
    - Configuration:
      - Uses OpenAI API credential.
      - Model set to "gpt-4.1-mini".
    - Inputs: From AI Agent (prompt messages).
    - Outputs: Responses passed back to AI Agent.
    - Edge Cases:
      - API key expiration or invalid.
      - Model availability or quota issues.
    - Version: 1.2.

  - **Calculator**
    - Type: LangChain Calculator tool node
    - Role: Performs precise percentage and arithmetic calculations as requested by AI Agent.
    - Configuration: No user parameters; invoked via AI Agent.
    - Inputs/Outputs: Connected bi-directionally with AI Agent as a tool.
    - Edge Cases:
      - Calculation errors if input data invalid.
    - Version: 1.

  - **Think**
    - Type: LangChain Think tool node
    - Role: AI reasoning and validation tool, used by AI Agent for internal checks.
    - Configuration: No user parameters; invoked by AI Agent.
    - Inputs/Outputs: Connected as AI tool with AI Agent.
    - Edge Cases:
      - AI reasoning failures if input ambiguous.
    - Version: 1.

---

#### 2.5 Email Dispatch

- **Overview:** Sends the generated HTML report to the client’s specified email address with a subject reflecting the client name and report period.

- **Nodes Involved:**
  - `Send a message`

- **Node Details:**

  - **Send a message**
    - Type: Gmail node (n8n-nodes-base.gmail)
    - Role: Email sending node delivering the HTML report.
    - Configuration:
      - Recipient email dynamically set from form input.
      - Email body set to AI Agent’s HTML output.
      - Email subject assembled as: "{Client Name} GA4 {Month name of Report from start date} Report".
      - Attribution appended: disabled.
      - Uses Gmail OAuth2 credential.
    - Inputs: From AI Agent node.
    - Outputs: Final node, sends email.
    - Edge Cases:
      - Gmail OAuth2 token expiration or lack of send permission.
      - Invalid recipient email address.
      - Large HTML size may cause email delivery issues.
    - Version: 2.1.

---

### 3. Summary Table

| Node Name                 | Node Type                                  | Functional Role                           | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                             |
|---------------------------|--------------------------------------------|-----------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| Get Client                | Form Trigger (n8n-nodes-base.formTrigger)  | Collects user input (GA4 details, dates, email) | None                         | Overall Metrics This Period  | Collects GA4 Property ID, key event, date ranges, client name, email. See setup checklist sticky note. |
| Overall Metrics This Period | HTTP Request                              | Fetches GA4 metrics for current period  | Get Client                   | Overall Metrics Previous Period | Must use Google Analytics OAuth2 credential.                                                         |
| Overall Metrics Previous Period | HTTP Request                         | Fetches GA4 metrics for previous period | Overall Metrics This Period  | Form Submits This Period      | Must use Google Analytics OAuth2 credential.                                                         |
| Form Submits This Period  | HTTP Request                              | Fetches key event counts current period | Overall Metrics Previous Period | Form Submits Previous Period | Must use Google Analytics OAuth2 credential.                                                         |
| Form Submits Previous Period | HTTP Request                           | Fetches key event counts previous period | Form Submits This Period      | Code                        | Must use Google Analytics OAuth2 credential.                                                         |
| Code                      | Code (JavaScript)                         | Normalizes date strings in form input   | Form Submits Previous Period | AI Agent                    | Removes hyphens from date fields for API compatibility.                                              |
| AI Agent                  | LangChain Agent                          | Generates full HTML report with AI insights | Code, Calculator, Think, OpenAI Chat Model | Send a message              | Uses AI, Calculator, Think tools to produce validated HTML report.                                   |
| OpenAI Chat Model          | LangChain OpenAI Chat Model              | Provides GPT-4.1 model for AI Agent     | AI Agent (prompt)             | AI Agent (response)          | Requires OpenAI API credential.                                                                      |
| Calculator                | LangChain Calculator tool                 | Performs arithmetic calculations         | AI Agent (tool)               | AI Agent (tool output)       | Used by AI Agent to do % change calculations.                                                       |
| Think                     | LangChain Think tool                      | AI reasoning and validation tool         | AI Agent (tool)               | AI Agent (tool output)       | Ensures data and HTML correctness in AI output.                                                     |
| Send a message            | Gmail Node (n8n-nodes-base.gmail)         | Sends final HTML report via email        | AI Agent                     | None                        | Uses Gmail OAuth2 credential. Sends to email specified in form.                                      |
| Sticky Note               | Sticky Note                              | Overview and project documentation       | None                         | None                        | Contains detailed workflow overview and setup instructions.                                         |
| Sticky Note1              | Sticky Note                              | Setup checklist                          | None                         | None                        | Setup checklist for credentials and test data.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Get Client"):**
   - Type: Form Trigger
   - Configure form fields:
     - "Account ID" (text, required)
     - "Key Event" (text, required)
     - "Report from start date" (date, required)
     - "Report to this end date" (date, required)
     - "Previous start date" (date, required)
     - "Previous end date" (date, required)
     - "Client Name" (text, required)
     - "Send to email:" (email, required)
   - Add introductory HTML explaining workflow purpose.
   - Ensure webhook ID is set or auto-generated.

2. **Create HTTP Request Node ("Overall Metrics This Period"):**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://analyticsdata.googleapis.com/v1beta/properties/{{ $json["Account ID"] }}:runReport`
   - Body (JSON):
     ```
     {
       "dateRanges": [{"startDate":"{{ $json['Report from start date'] }}","endDate":"{{ $json['Report to this end date'] }}"}],
       "metrics": [
         {"name":"sessions"},
         {"name":"engagedSessions"},
         {"name":"averageSessionDuration"},
         {"name":"bounceRate"},
         {"name":"screenPageViewsPerSession"},
         {"name":"eventsPerSession"},
         {"name":"newUsers"},
         {"name":"activeUsers"},
         {"name":"sessionKeyEventRate:{{ $json['Key Event'] }}"}
       ],
       "dimensions": []
     }
     ```
   - Headers: Content-Type: application/json
   - Authentication: Google Analytics OAuth2 credential required.
   - Connect `Get Client` main output to this node.

3. **Create HTTP Request Node ("Overall Metrics Previous Period"):**
   - Same as above, but for previous period dates.
   - Use "Previous start date" and "Previous end date" for dateRanges.
   - Connect from `Overall Metrics This Period` main output.

4. **Create HTTP Request Node ("Form Submits This Period"):**
   - Type: HTTP Request
   - Method: POST
   - URL: same GA4 endpoint with Account ID.
   - Body (JSON):
     ```
     {
       "dateRanges": [{"startDate":"{{ $json['Report from start date'] }}","endDate":"{{ $json['Report to this end date'] }}"}],
       "metrics": [{"name":"eventCount"}],
       "dimensions": [],
       "dimensionFilter": {
         "filter": {
           "fieldName": "eventName",
           "stringFilter": {
             "value": "{{ $json['Key Event'] }}",
             "matchType": "EXACT",
             "caseSensitive": false
           }
         }
       }
     }
     ```
   - Authentication: Google Analytics OAuth2.
   - Connect from `Overall Metrics Previous Period`.

5. **Create HTTP Request Node ("Form Submits Previous Period"):**
   - Same as above but for previous period dates.
   - Connect from `Form Submits This Period`.

6. **Create Code Node ("Code"):**
   - Type: Code (JavaScript)
   - Code:
     ```js
     const form = $('Get Client').first().json;
     const fields = ["Report from start date", "Report to this end date", "Previous start date", "Previous end date"];
     fields.forEach(field => {
       const val = form[field];
       if (val && typeof val === 'string') {
         form[field] = val.replace(/-/g, '');
       }
     });
     return [{ json: form }];
     ```
   - Connect from `Form Submits Previous Period`.

7. **Create LangChain Nodes for AI:**

   - **OpenAI Chat Model:**
     - Type: LangChain OpenAI Chat Model.
     - Set Model to "gpt-4.1-mini".
     - Attach OpenAI API credential.

   - **Calculator:**
     - Type: LangChain Calculator tool.
     - No parameters.

   - **Think:**
     - Type: LangChain Think tool.
     - No parameters.

   - **AI Agent:**
     - Type: LangChain Agent.
     - Parameters:
       - Provide prompt as per the detailed instructions from the original workflow, including:
         - Instructions for data inputs, percentage change rules, HTML structure, coloring conventions.
         - Embedding references to all relevant nodes’ data using expressions.
         - Max iterations: 50.
     - Connect:
       - `Code` main output to AI Agent main input.
       - AI Agent’s ai_languageModel input to `OpenAI Chat Model`.
       - AI Agent’s ai_tool input to both `Calculator` and `Think`.
     - This node generates the entire HTML report string.

8. **Create Gmail Node ("Send a message"):**
   - Type: Gmail node.
   - Authentication: Gmail OAuth2 credential.
   - Set recipient email: `={{ $('Get Client').item.json['Send to email:'] }}`
   - Set email subject: `={{ $('Get Client').item.json['Client Name'] }} GA4 {{ $('Get Client').item.json['Report from start date'].toDateTime().monthLong }} Report`
   - Set message body: `={{ $json.output }}` (output of AI Agent)
   - Disable attribution appended to message.
   - Connect `AI Agent` main output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates monthly GA4 KPI reporting with AI-generated insights and email delivery to specified recipients.                                                                                                              | Overview sticky note in workflow.                                                                                               |
| Setup requires Google Analytics OAuth2 credential, OpenAI API key credential, and Gmail OAuth2 credential with send permissions.                                                                                                | Setup checklist sticky note.                                                                                                    |
| GA4 Property ID guide: https://take.ms/vO2MG                                                                                                                                                                                    | Setup instructions sticky note.                                                                                                |
| Key Event naming convention guide: https://take.ms/hxwQi                                                                                                                                                                        | Setup instructions sticky note.                                                                                                |
| Google Analytics Data API documentation: https://developers.google.com/analytics/devguides/reporting/data/v1                                                                                                                   | For understanding API metrics and requests.                                                                                    |
| OpenAI credentials setup: https://docs.n8n.io/integrations/builtin/credentials/openai/                                                                                                                                           | Credential setup for AI nodes.                                                                                                 |
| Google OAuth2 (GA4) credentials setup: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/                                                                                                              | Credential setup for Google Analytics API.                                                                                    |
| Gmail OAuth2 credentials setup: https://docs.n8n.io/integrations/builtin/credentials/google/                                                                                                                                     | Credential setup for sending email via Gmail.                                                                                  |
| Use strict prompt instructions to avoid AI hallucination; always verify data presence before generating recommendations.                                                                                                         | AI Agent prompt instructions.                                                                                                 |
| Percentage changes coloring: positive changes in green (#10B981), negative changes in red (#EF4444), with the exception of bounce rate which inverts logic for color application.                                                | AI Agent prompt and HTML styling guidelines.                                                                                  |
| The final HTML email uses responsive design and inline styles for compatibility across email clients; includes VertoDigital branding and Google Analytics links for transparency and exploration.                               | HTML email template embedded in AI Agent prompt.                                                                              |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created in n8n, an integration and automation tool. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.