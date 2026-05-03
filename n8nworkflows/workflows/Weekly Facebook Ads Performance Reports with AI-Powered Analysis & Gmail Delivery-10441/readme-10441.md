Weekly Facebook Ads Performance Reports with AI-Powered Analysis & Gmail Delivery

https://n8nworkflows.xyz/workflows/weekly-facebook-ads-performance-reports-with-ai-powered-analysis---gmail-delivery-10441


# Weekly Facebook Ads Performance Reports with AI-Powered Analysis & Gmail Delivery

---

## 1. Workflow Overview

This workflow automates the generation and delivery of weekly Facebook Ads performance reports enhanced with AI-driven analysis. It is designed for marketing teams or agencies that want to save time on manual data extraction, analysis, and reporting tasks by automating the entire reporting pipeline.

**Use Cases:**
- Weekly automated Facebook Ads reporting with AI-generated insights.
- Automated PDF report generation and email delivery.
- Scalable for multiple campaigns with detailed per-campaign analytics.
- Enables data-driven decision-making with actionable recommendations.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Data Fetching**  
  The workflow kicks off every Monday morning, securely fetching the previous week's Facebook Ads campaign data via Facebook Graph API.

- **1.2 Data Formatting & Per-Campaign AI Summarization**  
  Raw Facebook data is flattened and formatted. Each campaign’s data is sent to an AI agent specialized in marketing analysis to generate individual campaign performance summaries and actionable insights.

- **1.3 Merging Campaign Summaries & Final AI Report Writing**  
  All individual campaign summaries are aggregated and passed to a second AI model that produces a cohesive, client-friendly, fully formatted HTML report in compliance with strict formatting rules.

- **1.4 PDF Generation & Email Delivery**  
  The HTML report is converted into a PDF file via a third-party PDF generation service (PDFCrowd) and emailed automatically to clients via Gmail with a professional message.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Trigger & Data Fetching

**Overview:**  
This block initiates the workflow on a weekly schedule and pulls Facebook Ads data for the last 7 days from the Facebook Graph API.

**Nodes Involved:**  
- Weekly Trigger  
- Fetch FB Data  
- Format Data

#### Node: Weekly Trigger  
- **Type:** Schedule Trigger  
- **Role:** Starts workflow every Monday at 8:30 AM.  
- **Configuration:**  
  - Interval: Weekly  
  - Trigger Day: Monday (Day 1)  
  - Trigger Time: 08:30 hrs  
- **Input/Output:** No input; output triggers next node.  
- **Failures:** Timezone misconfiguration or scheduling issues can delay trigger.  
- **Version:** 1.2

#### Node: Fetch FB Data  
- **Type:** HTTP Request  
- **Role:** Calls Facebook Graph API to fetch ad insights for the last 7 days.  
- **Configuration:**  
  - GET to URL: `https://graph.facebook.com/v20.0/act_{{Your_Ad_Account_ID}}/insights`  
  - Fields: campaign_name, impressions, clicks, spend, actions  
  - Date preset: last_7d  
  - Auth: Facebook access token in query and header.  
- **Input:** Trigger from Weekly Trigger  
- **Output:** Raw Facebook data JSON array.  
- **Failures:**  
  - API errors (invalid token, rate-limiting, permissions)  
  - Network timeouts  
  - Incorrect account ID or token  
- **Version:** 4.2  
- **Sticky Note:** "## Fetch & Format Data"

#### Node: Format Data  
- **Type:** Code (JavaScript)  
- **Role:** Flattens raw Facebook API response into simplified campaign objects for easier processing.  
- **Configuration:**  
  - Maps each item, converts string numbers to numbers, extracts action values by type (e.g., link_click, purchase).  
- **Key Expression:** Custom JS code mapping fields and action arrays to flat objects.  
- **Input:** Output from Fetch FB Data  
- **Output:** Array of campaign summaries with typed metrics.  
- **Failures:** Code errors if unexpected data structure or missing fields.  
- **Version:** 2  
- **Sticky Note:** "## Fetch & Format Data"

---

### 2.2 Data Summarization per Campaign with AI

**Overview:**  
Each formatted campaign data item is sent to an AI agent that analyzes performance metrics and generates a clear, concise, client-friendly summary including insights and recommendations.

**Nodes Involved:**  
- AI Summarizer

#### Node: AI Summarizer  
- **Type:** Langchain Agent (OpenAI)  
- **Role:** Performs detailed analysis on a single campaign’s data and outputs a structured textual summary.  
- **Configuration:**  
  - System message defines role as marketing analyst specializing in Facebook Ads reporting.  
  - Input text template injects campaign fields: campaign_name, impressions, clicks, spend, date range, purchases, link clicks.  
  - Output includes sections: campaign name, date range, insights (engagement, traffic, conversions, strengths/weaknesses), and 3-5 actionable recommendations.  
  - Tone: concise, client-friendly, no jargon.  
- **Input:** Each campaign JSON item from Format Data  
- **Output:** Campaign-specific performance summary text.  
- **Failures:**  
  - API errors (key invalid, rate limits)  
  - Prompt injection or variable substitution errors  
  - Unexpected data formats causing incomplete summaries  
- **Version:** 2.2  
- **Sticky Note:** "## Summarize Campaign Performance"

---

### 2.3 Merging Summaries and Final AI Report Composition

**Overview:**  
Aggregates all individual campaign summaries into a single array and passes them to a second AI model to create a polished HTML report with a specific formatting standard.

**Nodes Involved:**  
- Merge Reports  
- AI Report Writer  
- Set Node

#### Node: Merge Reports  
- **Type:** Aggregate  
- **Role:** Combines all individual campaign summary texts into a single collection for final report generation.  
- **Configuration:** Aggregate all "output" fields into an array.  
- **Input:** Outputs from AI Summarizer (one per campaign)  
- **Output:** Array of summaries  
- **Failures:** Empty input array could cause errors downstream.  
- **Version:** 1

#### Node: AI Report Writer  
- **Type:** Langchain OpenAI Chat Model  
- **Role:** Generates the final client report in valid HTML5 format, combining all campaign summaries and adding overall analysis and recommendations.  
- **Configuration:**  
  - Model: GPT-4.1-mini  
  - System prompt instructs to produce a valid HTML5 document with specific tags only (h1, h2, h3, p, ul, li, etc.)  
  - Input: JSON array of campaign summaries from Merge Reports  
  - Output: Full HTML report string.  
- **Input:** Combined campaign summaries array  
- **Output:** HTML report string  
- **Failures:** API errors, prompt formatting issues, invalid HTML output  
- **Version:** 1.8  
- **Sticky Note:** "## Generate Final Performance Report"

#### Node: Set Node  
- **Type:** Set  
- **Role:** Extracts the HTML content from AI Report Writer output to prepare for PDF generation.  
- **Configuration:** Assigns `html` property from AI Report Writer’s message content.  
- **Input:** Output from AI Report Writer  
- **Output:** JSON with `html` property for PDF conversion  
- **Failures:** If AI output structure changes, this assignment may fail.  
- **Version:** 3.4

---

### 2.4 PDF Generation and Email Sending

**Overview:**  
Converts the generated HTML report into a PDF file and emails it to the client via Gmail with a professional message and the PDF attached.

**Nodes Involved:**  
- Generate PDF  
- Send Email

#### Node: Generate PDF  
- **Type:** HTTP Request  
- **Role:** Sends the HTML report to PDFCrowd API to convert HTML into a PDF document.  
- **Configuration:**  
  - POST to PDFCrowd API endpoint  
  - Content-Type: form-urlencoded  
  - Body parameter: `text` contains the HTML report  
  - Auth: Basic Auth header with PDFCrowd API key  
  - Response: Binary file PDF saved under a named property for attachment  
- **Input:** HTML report from Set Node  
- **Output:** PDF binary data for next node  
- **Failures:**  
  - API key invalid or quota exceeded  
  - HTML input too large or malformed  
  - Network issues  
- **Version:** 4.2  
- **Sticky Note:** "## Generate & Share PDF"

#### Node: Send Email  
- **Type:** Gmail node  
- **Role:** Sends an email with the weekly Facebook Ads performance report attached as PDF.  
- **Configuration:**  
  - Subject: Weekly Facebook Ads Report with date range  
  - Message body: Personalized text with highlights and friendly tone  
  - Attachment: PDF file from previous node  
  - Credentials: Gmail OAuth2 setup in n8n credentials  
- **Input:** PDF from Generate PDF  
- **Output:** Email sent confirmation  
- **Failures:**  
  - Authentication errors (OAuth token expired)  
  - Attachment size limits or network errors  
  - Incorrect email address configuration  
- **Version:** 2.1

---

## 3. Summary Table

| Node Name       | Node Type                        | Functional Role                              | Input Node(s)      | Output Node(s)         | Sticky Note                                                                                  |
|-----------------|---------------------------------|----------------------------------------------|--------------------|------------------------|----------------------------------------------------------------------------------------------|
| Weekly Trigger  | Schedule Trigger                | Weekly schedule trigger to start workflow    | -                  | Fetch FB Data          |                                                                                              |
| Fetch FB Data   | HTTP Request                   | Fetch Facebook Ads campaign data for last 7 days | Weekly Trigger    | Format Data            | ## Fetch & Format Data                                                                       |
| Format Data     | Code (JavaScript)              | Flatten and convert raw FB data for AI input | Fetch FB Data      | AI Summarizer          | ## Fetch & Format Data                                                                       |
| AI Summarizer   | Langchain Agent (OpenAI)       | Generate per-campaign performance summaries  | Format Data        | Merge Reports          | ## Summarize Campaign Performance                                                           |
| Merge Reports   | Aggregate                      | Combine all campaign summaries into array    | AI Summarizer      | AI Report Writer       |                                                                                              |
| AI Report Writer| Langchain OpenAI Chat Model    | Create final HTML report from campaign summaries | Merge Reports    | Set Node               | ## Generate Final Performance Report                                                        |
| Set Node        | Set                           | Extract HTML content for PDF generation       | AI Report Writer   | Generate PDF           |                                                                                              |
| Generate PDF    | HTTP Request                   | Convert HTML report to PDF via PDFCrowd API  | Set Node           | Send Email             | ## Generate & Share PDF                                                                      |
| Send Email      | Gmail                         | Email the PDF report to client                 | Generate PDF       | -                      |                                                                                              |
| Sticky Note     | Sticky Note                   | Informational notes                            | -                  | -                      | ## Facebook Ads Weekly Performance Report Generator; Explains workflow purpose, usage, and requirements |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run weekly every Monday at 8:30 AM (local timezone as required).

2. **Add HTTP Request node named "Fetch FB Data"**  
   - Method: GET  
   - URL: `https://graph.facebook.com/v20.0/act_{{Your_Ad_Account_ID}}/insights?fields=campaign_name,impressions,clicks,spend,actions&date_preset=last_7d&access_token={{Your_FB_Access_Token}}`  
   - Add header parameter `Auth` with value `={{FACEBOOK_ACCESS_TOKEN}}` or include access token in URL query string.  
   - Connect Schedule Trigger → Fetch FB Data.

3. **Add Code node named "Format Data"**  
   - Paste JS code to map and flatten the Facebook API response into plain JSON objects with typed numbers and action metrics (see below):  
   ```js
   return items.map(item => {
       const data = item.json;
       const flattened = {
           campaign_name: data.campaign_name,
           impressions: Number(data.impressions),
           clicks: Number(data.clicks),
           spend: Number(data.spend),
           date_start: data.date_start,
           date_stop: data.date_stop
       };
       if (Array.isArray(data.actions)) {
           data.actions.forEach(action => {
               flattened[action.action_type] = Number(action.value);
           });
       }
       return { json: flattened };
   });
   ```
   - Connect Fetch FB Data → Format Data.

4. **Add Langchain Agent node "AI Summarizer"**  
   - Provider: OpenAI with GPT-4.1-mini model  
   - Configure system message as marketing analyst for Facebook Ads, instructing to output a concise, actionable analysis for one campaign with specified fields.  
   - Input text template to include the campaign data parameters from Format Data outputs.  
   - Connect Format Data → AI Summarizer.

5. **Add Aggregate node "Merge Reports"**  
   - Aggregate all outputs from AI Summarizer into a single array (aggregate on “output” field).  
   - Connect AI Summarizer → Merge Reports.

6. **Add Langchain OpenAI Chat node "AI Report Writer"**  
   - Model: GPT-4.1-mini  
   - System message instructing to produce a valid HTML5 report document using only specified tags, combining all campaign summaries for a weekly report.  
   - Input: the aggregated array from Merge Reports.  
   - Connect Merge Reports → AI Report Writer.

7. **Add Set node "Set Node"**  
   - Assign a new field called `html` with the value extracted from AI Report Writer output message content.  
   - Connect AI Report Writer → Set Node.

8. **Add HTTP Request node "Generate PDF"**  
   - Method: POST  
   - URL: PDFCrowd API endpoint (e.g., `https://api.pdfcrowd.com/convert/24.04`)  
   - Headers: Basic Authorization with PDFCrowd API key  
   - Body: form-urlencoded with parameter `text` set to the HTML content from Set Node's `html` field  
   - Configure response to capture binary PDF file with a meaningful filename property.  
   - Connect Set Node → Generate PDF.

9. **Add Gmail node "Send Email"**  
   - Configure Gmail OAuth2 credentials in n8n credentials manager.  
   - Subject: “Weekly Facebook Ads Report | [Date Range]”  
   - Message body: Professional and friendly message including quick highlights and sign-off.  
   - Attach the PDF binary output from Generate PDF node.  
   - Connect Generate PDF → Send Email.

10. **Test workflow end-to-end**  
    - Verify Facebook API access tokens and permissions  
    - Verify OpenAI API keys and prompt correctness  
    - Verify PDFCrowd API key and quota  
    - Verify Gmail credentials and sending permissions

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| **Workflow Description:** This automation generates a professional weekly Facebook Ads performance report by fetching last 7 days data, using AI to analyze and summarize each campaign, compiling an HTML report, converting it to PDF, and emailing it to clients automatically. It saves hours of manual reporting for marketing teams. Requires Facebook Graph API access token, OpenAI API key, PDFCrowd API key, and Gmail credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Workflow internal sticky note explaining purpose and requirements.                                               |
| **Formatting Guidelines for AI Report:** The final report uses only HTML5 tags h1, h2, h3, p, ul, li, strong, em, and optionally table elements for clarity. Avoids Markdown, scripts, styles, or complex HTML. This ensures compatibility with PDF generation and email clients.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | System prompt in AI Report Writer node.                                                                           |
| **API Security Recommendations:** Store all API tokens and keys securely in n8n credentials manager. Avoid hardcoding tokens directly in nodes. Use environment variables or credential references for security best practices.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Best practice note for implementation security.                                                                  |
| **Customization Tips:** The time period (date range), email content, and PDF design can be customized by editing the schedule trigger, modifying the email node text, and adjusting the HTML output template or PDFCrowd parameters respectively.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky note advice in workflow.                                                                                    |
| **Credits and Additional Resources:** The workflow was authored by Muhammad Ali Zubair, AI Automation Expert. For further insights on automating marketing reports, consider reading external blogs on Facebook Ads API and AI-powered reporting tools.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Author credit and suggestion for further learning.                                                                |

---

# Disclaimer

This document is based exclusively on an automated workflow created with n8n, respecting all applicable content policies, containing no illegal or protected material. All processed data is legal and publicly accessible.

---