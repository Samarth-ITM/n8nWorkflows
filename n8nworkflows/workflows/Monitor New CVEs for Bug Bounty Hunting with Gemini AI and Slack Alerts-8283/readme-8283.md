Monitor New CVEs for Bug Bounty Hunting with Gemini AI and Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-new-cves-for-bug-bounty-hunting-with-gemini-ai-and-slack-alerts-8283


# Monitor New CVEs for Bug Bounty Hunting with Gemini AI and Slack Alerts

### 1. Workflow Overview

This workflow, titled **"Monitor New CVEs for Bug Bounty Hunting with Gemini AI and Slack Alerts"**, automates the monitoring of newly published Common Vulnerabilities and Exposures (CVEs) from the NIST database. Its primary use case is to assist bug bounty hunters by filtering and analyzing recent CVEs using Google Gemini AI to generate actionable, Slack-formatted relevance assessments, then posting these assessments to a Slack channel.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled CVE Retrieval:** Periodically triggers and fetches the latest CVEs published in the last hour from the NIST public API.
- **1.2 CVE Data Processing:** Parses and formats key CVE information such as CVE ID, publication date, severity scores, descriptions, and references.
- **1.3 AI-Powered CVE Analysis:** Uses Google Gemini AI via LangChain nodes to analyze the CVE data and generate concise, bug bounty relevance assessments.
- **1.4 Slack Notification:** Sends the formatted AI-generated CVE assessments to a configured Slack channel via a Slack bot.

Supporting sticky notes document setup instructions and explanations for each functional block.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled CVE Retrieval

- **Overview:**  
  This block triggers the workflow every hour and requests the latest CVEs published in the last hour from the NIST API, limited to 20 results.

- **Nodes Involved:**  
  - Schedule Trigger  
  - HTTP Request  
  - Split Out  
  - 📒 NIST API (sticky note)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Trigger node  
    - *Role:* Initiates the workflow every hour.  
    - *Configuration:* Interval set to every 1 hour.  
    - *Input:* None (trigger node)  
    - *Output:* Starts workflow execution  
    - *Edge cases:* Misconfiguration could cause no triggering or too frequent triggering.  
    - *Notes:* None

  - **HTTP Request**  
    - *Type:* HTTP GET request  
    - *Role:* Calls NIST CVE API to fetch CVEs published in the last hour.  
    - *Configuration:*  
      - URL: `https://services.nvd.nist.gov/rest/json/cves/2.0`  
      - Query Parameters:  
        - `pubStartDate`: ISO timestamp one hour ago (dynamic expression)  
        - `pubEndDate`: Current ISO timestamp  
        - `resultsPerPage`: 20 (max results per run)  
        - `startIndex`: 0 (pagination start)  
      - No authentication required (public API)  
    - *Input:* Trigger from Schedule Trigger  
    - *Output:* JSON response containing CVE data  
    - *Edge cases:* API downtime, network errors, malformed date expressions, empty results.  
    - *Sticky Note:* 📒 NIST API - Documents API usage and limits.

  - **Split Out**  
    - *Type:* Split Out node  
    - *Role:* Splits the array of vulnerabilities into individual CVE items for processing.  
    - *Configuration:* Field to split out: `vulnerabilities` (array from API response)  
    - *Input:* Output from HTTP Request  
    - *Output:* Emits one CVE item per execution  
    - *Edge cases:* Empty or missing `vulnerabilities` field could cause no output or errors.  
    - *Notes:* None

#### 2.2 CVE Data Processing

- **Overview:**  
  Extracts and formats key CVE fields (ID, publication date, severity, descriptions, references) into a simplified structure suitable for AI input.

- **Nodes Involved:**  
  - Edit Fields  
  - 📒 Processing (sticky note)

- **Node Details:**

  - **Edit Fields**  
    - *Type:* Set node  
    - *Role:* Extracts from each CVE JSON object the following fields, formatting dates and severity for readability:  
      - `cve`: CVE ID (string)  
      - `published`: Formatted publication date in UTC, e.g., "Wed, 27 Sep 2023, 14:00 (UTC)"  
      - `cve_descriptions`: First CVE description text  
      - `severity`: Extracts CVSS metrics (v4.0, v3.1, v3.0, v2) with severity level and base score, falls back to "Unknown" if none found  
      - `references`: First reference URL  
    - *Input:* Single CVE object from Split Out  
    - *Output:* Structured JSON for AI analysis  
    - *Key expressions:*  
      - Date formatting uses JavaScript `toLocaleString` with UTC timezone  
      - Severity evaluation cascades through CVSS versions  
    - *Edge cases:* Missing descriptions, references, or CVSS metrics; could yield empty or "Unknown" fields.  
    - *Sticky Note:* 📒 Processing - Explains which fields are extracted and why.

#### 2.3 AI-Powered CVE Analysis

- **Overview:**  
  Sends the processed CVE data to Google Gemini AI (via LangChain integration) to generate a concise, bug bounty relevance analysis formatted for Slack.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - CVE Summarizer  
  - 📒 AI Analysis (sticky note)

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type:* AI Language Model (Google Gemini via LangChain)  
    - *Role:* Provides the underlying large language model for generating CVE analysis.  
    - *Configuration:* Uses model `"models/gemini-2.5-pro"`  
    - *Credentials:* Google Palm API credentials labeled "Google Gemini Cheekymisa"  
    - *Input:* Receives prompt from CVE Summarizer node (as AI language model input)  
    - *Output:* AI-generated text response with CVE assessment  
    - *Edge cases:* API rate limits, credential misconfiguration, model unavailability or latency.

  - **CVE Summarizer**  
    - *Type:* LangChain Agent node  
    - *Role:* Constructs the system and user prompt for AI to analyze CVE relevance, focusing on bug bounty applicability.  
    - *Configuration:*  
      - Dynamic prompt text compiling CVE ID, severity, published date, description, and references  
      - System message instructs the AI to produce Slack-ready, concise, actionable bug bounty relevance assessments, including a strict formatting guideline and examples.  
      - Output limited to 5 lines, Slack hyperlink formatting, and explicit relevance categories (HIGH/MEDIUM/LOW/NONE).  
    - *Input:* Structured CVE data from Edit Fields  
    - *Output:* Prompt to AI, then AI response with formatted assessment  
    - *Edge cases:* Expression errors in prompt building, incomplete data, AI hallucination or irrelevant output.  
    - *Sub-workflow:* Integrates tightly with Google Gemini Chat Model node.  
    - *Sticky Note:* 📒 AI Analysis - Setup and usage instructions for Google Gemini API.

#### 2.4 Slack Notification

- **Overview:**  
  Sends the AI-generated CVE assessment messages to a designated Slack channel using a Slack bot.

- **Nodes Involved:**  
  - Send a message  
  - 📒 Slack Setup (sticky note)

- **Node Details:**

  - **Send a message**  
    - *Type:* Slack node  
    - *Role:* Posts the AI-generated CVE assessment text to the configured Slack channel.  
    - *Configuration:*  
      - Message text set dynamically from AI output JSON field `output`.  
      - Channel selected via credential-stored channel ID (list mode with cached channel names).  
      - Option to disable "link to workflow" included (disabled).  
    - *Credentials:* Slack API credentials named "Slack Whisper bot" with appropriate bot token and scopes.  
    - *Input:* AI-generated text from CVE Summarizer  
    - *Output:* None (final node)  
    - *Edge cases:* Slack API rate limit errors, invalid or missing channel ID, credential expiration.  
    - *Sticky Note:* 📒 Slack Setup - Setup instructions for Slack app, bot token scopes, and channel configuration.

---

### 3. Summary Table

| Node Name              | Node Type                                 | Functional Role                  | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                     |
|------------------------|-------------------------------------------|---------------------------------|------------------------|-----------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Trigger                                   | Periodic execution trigger       | —                      | HTTP Request           |                                                                                                                |
| HTTP Request          | HTTP Request                              | Fetches recent CVEs from NIST API | Schedule Trigger        | Split Out              | 📒 NIST API: Public API, no credentials, max 20 results, hourly fetch                                          |
| Split Out             | Split Out                                 | Splits multiple CVEs into single items | HTTP Request           | Edit Fields            |                                                                                                                |
| Edit Fields           | Set                                       | Extracts and formats CVE key fields | Split Out               | CVE Summarizer         | 📒 Processing: Extracts CVE ID, date, severity, description, references for AI input                           |
| Google Gemini Chat Model | AI Language Model (LangChain)             | Provides AI model for CVE analysis | CVE Summarizer (ai_languageModel) | CVE Summarizer (main) | 📒 AI Analysis: Google Gemini API setup and usage instructions                                               |
| CVE Summarizer        | LangChain Agent                           | Constructs AI prompt and interprets AI output | Edit Fields            | Send a message         | 📒 AI Analysis: Detailed prompt for bug bounty relevance; Slack formatting, actionable insights                |
| Send a message        | Slack                                     | Sends AI CVE assessment to Slack  | CVE Summarizer          | —                     | 📒 Slack Setup: Slack app creation, bot token scopes, credentials, target channel configuration                |
| 📒 Overview           | Sticky Note                               | General project overview          | —                      | —                     | Describes workflow purpose and setup steps                                                                    |
| 📒 NIST API           | Sticky Note                               | Documents NIST API usage          | —                      | —                     | Public API, no rate limits, hourly CVE fetch                                                                   |
| 📒 Processing         | Sticky Note                               | Explains CVE data processing      | —                      | —                     | Extracts CVE details for AI input                                                                               |
| 📒 AI Analysis        | Sticky Note                               | Details AI analysis setup and features | —                      | —                     | Setup for Google Gemini API, AI features, output formatting                                                   |
| 📒 Slack Setup        | Sticky Note                               | Slack integration setup instructions | —                      | —                     | Slack app creation, bot token scopes, install, channel configuration                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node:**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 1 hour.

2. **Create "HTTP Request" node:**  
   - Type: HTTP Request  
   - URL: `https://services.nvd.nist.gov/rest/json/cves/2.0`  
   - Method: GET  
   - Query parameters:  
     - `pubStartDate`: Expression → `={{ new Date(Date.now() - 60 * 60 * 1000).toISOString() }}`  
     - `pubEndDate`: Expression → `={{ new Date().toISOString() }}`  
     - `resultsPerPage`: 20  
     - `startIndex`: 0  
   - No authentication needed.  
   - Connect Schedule Trigger output to this node input.

3. **Create "Split Out" node:**  
   - Type: Split Out  
   - Field to split out: `vulnerabilities` (from HTTP Request response)  
   - Connect HTTP Request output to Split Out input.

4. **Create "Edit Fields" (Set) node:**  
   - Type: Set  
   - Add fields with expressions:  
     - `cve`: `={{ $json.cve.id }}`  
     - `published`: `={{ new Date($json.cve.published).toLocaleString('en-GB', { timeZone: 'UTC', weekday: 'short', year: 'numeric', month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit', hour12: false }) + ' (UTC)' }}`  
     - `cve_descriptions`: `={{ $json.cve.descriptions[0].value }}`  
     - `severity`: Use cascading ternary expression to check CVSS v4.0, v3.1, v3.0, v2 metrics and format `"SEVERITY (SCORE)"` or fallback `"Unknown"`  
     - `references`: `={{ $json.cve.references[0].url }}`  
   - Connect Split Out output to Edit Fields input.

5. **Create "Google Gemini Chat Model" node:**  
   - Type: LangChain AI Language Model (Google Gemini)  
   - Model name: `models/gemini-2.5-pro`  
   - Credentials: Create Google Palm API credentials and assign here.  
   - Connect this node to the `ai_languageModel` input of the next node (CVE Summarizer).

6. **Create "CVE Summarizer" node:**  
   - Type: LangChain Agent  
   - Text parameter:  
     ```
     CVE: {{ $json.cve }}
     Severity: {{ $json.severity }}
     Published: {{ $json.published }}
     Description: {{ $json.cve_descriptions }}
     Ref: {{ $json.references }}
     ```
   - System message: Provide detailed instructions for AI to generate Slack-formatted bug bounty relevance assessments, including output format, mindset, and style as per the prompt in the original workflow.  
   - Connect Edit Fields output to main input of this node.  
   - Connect Google Gemini Chat Model output to `ai_languageModel` input of this node.

7. **Create "Send a message" node:**  
   - Type: Slack node  
   - Text: `={{ $json.output }}` (AI-generated CVE assessment)  
   - Choose channel mode "list" and select the channel ID matching your Slack workspace.  
   - Credentials: Set up Slack API credentials with bot token having scopes: `chat:write`, `channels:read`.  
   - Connect CVE Summarizer output to this node input.

8. **Add informative Sticky Notes:**  
   - Overview note describing workflow purpose and setup steps (Google Gemini API, Slack webhook, channel ID).  
   - NIST API note explaining public API usage, no credentials needed, limits.  
   - Processing note explaining data extraction rationale.  
   - AI Analysis note describing Google Gemini setup and output goals.  
   - Slack Setup note describing Slack app creation, bot token scopes, installation, and channel configuration.

9. **Validate all connections:**  
   - Schedule Trigger → HTTP Request → Split Out → Edit Fields → CVE Summarizer → Send a message  
   - Google Gemini Chat Model linked as AI model to CVE Summarizer.

10. **Test execution and monitor for errors:**  
    - Ensure Google Gemini and Slack credentials are valid and active.  
    - Confirm Slack channel ID is correct and accessible by the bot.  
    - Check API responses for empty or malformed CVE data.  
    - Adjust error handling or add retries if needed for production use.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Google Gemini API key can be obtained from https://aistudio.google.com/                                                                                             | AI provider and credential setup                     |
| Slack app creation guide: https://api.slack.com/apps — requires bot token with scopes `chat:write` and `channels:read`                                             | Slack integration setup                              |
| CVE relevance AI prompt designed for elite bug bounty hunters to focus on actionable testing strategies and modular exploitation techniques                      | AI prompt design and mindset                         |
| NIST CVE API is public with no rate limits and provides detailed CVE metrics including CVSS versions 2 to 4.                                                       | CVE data source                                     |
| Slack formatting rules used in AI outputs: Slack hyperlink `<https://example.com|Display Text>`, bold with `*`, max 5 lines per message                              | Formatting for Slack notifications                   |

---

This documentation enables full understanding, manual recreation, and future modification of the CVE monitoring workflow, ensuring robust integration between NIST CVE data, Google Gemini AI analysis, and Slack notifications tailored for bug bounty hunting purposes.

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal, offensive, or protected elements. All processed data is legal and public._