Generate Weekly Meta Ads Performance Reports with GPT-4 Insights and Slack Delivery

https://n8nworkflows.xyz/workflows/generate-weekly-meta-ads-performance-reports-with-gpt-4-insights-and-slack-delivery-5386


# Generate Weekly Meta Ads Performance Reports with GPT-4 Insights and Slack Delivery

### 1. Workflow Overview

This workflow automates the generation and delivery of weekly Meta Ads performance reports enriched with AI-generated insights. It is designed to extract advertising campaign data from Meta (Facebook) Ads API, process and aggregate the data, generate concise AI-driven insights at multiple ad hierarchy levels (campaign, ad set, ad), create a professional PDF report via pdforge, and deliver the report directly to a Slack channel. The workflow runs on a schedule (every Monday at 8 a.m.) but can be adapted for other triggers and delivery methods such as email or Telegram.

**Target Use Cases:**

- Marketing teams needing automated weekly summaries of Meta Ads performance
- Data analysts seeking AI-generated actionable insights on campaign efficiency
- Businesses wanting seamless report delivery into Slack or other communication tools

**Logical Blocks:**

- 1.1 Scheduled Trigger & Configuration  
- 1.2 Meta Ads Data Retrieval  
- 1.3 Data Splitting & Formatting  
- 1.4 AI Insight Generation  
- 1.5 Variables Preparation for Report  
- 1.6 PDF Report Generation via pdforge  
- 1.7 Report Download and Slack Delivery  

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration

**Overview:**  
This block initiates the workflow weekly on Monday at 8 a.m. It sets essential parameters such as the Meta Ads account ID and the date range for data retrieval.

**Nodes Involved:**  
- Weekly at monday 8am  
- Configuration

**Node Details:**

- **Weekly at monday 8am**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow every Monday at 8 a.m. (timezone America/Sao_Paulo)  
  - *Config:* Weekly interval with day=Monday, hour=8  
  - *Input:* None (trigger node)  
  - *Output:* Triggers Configuration node  
  - *Edge cases:* Missed triggers if n8n instance is down at trigger time  

- **Configuration**  
  - *Type:* Set node  
  - *Role:* Defines static workflow parameters  
  - *Config:* Sets `ad_account_id` to `982086037409625232` and `date_range` to `last_7d` (last 7 days)  
  - *Input:* Trigger from Weekly at monday 8am  
  - *Output:* Passes parameters to HTTP Request - Meta Ads node  
  - *Edge cases:* Hardcoded parameters require manual update for different accounts or periods

---

#### 1.2 Meta Ads Data Retrieval

**Overview:**  
Fetches detailed ad performance data from the Facebook Graph API for the specified ad account and date range, with granular daily data at the ad level.

**Nodes Involved:**  
- HTTP Request - Meta Ads  
- Split Out

**Node Details:**

- **HTTP Request - Meta Ads**  
  - *Type:* HTTP Request  
  - *Role:* Queries Facebook Graph API v23.0 for ad insights  
  - *Config:*  
    - URL built dynamically using `ad_account_id`  
    - Query parameters include fields like campaign, ad set, ad IDs/names, impressions, clicks, spend, CPC, ROAS  
    - `time_increment=1` to get daily data  
    - `date_preset` dynamically set to configured date range  
    - Level set to `ad` (granular data)  
    - Uses HTTP Header Auth with Meta Ads credentials  
    - Pagination enabled to fetch all pages until no next page is present  
  - *Input:* Configuration node output  
  - *Output:* Raw JSON data containing daily ad insights  
  - *Edge cases:*  
    - API rate limits or authentication failures  
    - Pagination errors or incomplete data if API changes  
    - Network timeouts handled by “continueRegularOutput” on error  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the array in the `data` field into individual items for further processing  
  - *Config:* Field to split out is `data` array from previous response  
  - *Input:* HTTP Request - Meta Ads node output  
  - *Output:* Individual ad insight records passed one by one downstream  
  - *Edge cases:* Empty or malformed `data` array may cause no outputs or errors

---

#### 1.3 Data Splitting & Formatting

**Overview:**  
Aggregates raw ad data records into structured summary statistics and prepared datasets for reporting, including overall totals, daily trends, and top performers.

**Nodes Involved:**  
- Format data

**Node Details:**

- **Format data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes all individual ad records and aggregates performance metrics at multiple levels  
  - *Config:*  
    - Extracts date ranges  
    - Aggregates impressions, clicks, spend, CTR, CPC globally and by campaign/ad set/ad  
    - Generates daily time series arrays for impressions and clicks  
    - Selects top 10 campaigns, ad sets, and ads by cheapest CPC  
    - Prepares sorted comparisons by investment descending  
    - Outputs a comprehensive JSON report object with all aggregated data  
  - *Input:* Split Out node outputs (all ad records)  
  - *Output:* Single JSON object representing the fully aggregated report data  
  - *Edge cases:*  
    - Handles missing numeric fields by defaulting to zero  
    - Date parsing assumes valid ISO date strings  
    - Large datasets may impact execution time or memory usage

---

#### 1.4 AI Insight Generation

**Overview:**  
Uses GPT-4 via LangChain agent to create concise, actionable insights from the aggregated Meta Ads data at campaign, ad set, and ad levels.

**Nodes Involved:**  
- Generating Insights for Meta Ads  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **Generating Insights for Meta Ads**  
  - *Type:* LangChain Agent  
  - *Role:* Orchestrates AI prompt to generate structured insights JSON from input data  
  - *Config:*  
    - Input text is JSON stringified aggregated report data (`Format data` output)  
    - System message defines persona, objectives, rules for insight text (200-300 chars each, JSON-only output)  
    - Parses AI output with Structured Output Parser downstream  
  - *Input:* Output of Format data node  
  - *Output:* Raw AI-generated insight JSON string  
  - *Edge cases:*  
    - AI may fail or return invalid JSON — mitigated by Structured Output Parser  
    - API rate limits or auth failures  
    - Length constraints enforced in prompt to keep insights concise  

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model node  
  - *Role:* Provides GPT-4.1 language model for the agent  
  - *Config:* Model set to `gpt-4.1` with default options  
  - *Input:* From Generating Insights for Meta Ads (agent’s language model input)  
  - *Output:* AI response for insight generation  
  - *Edge cases:* API key validity, model availability, network issues  

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser  
  - *Role:* Ensures the AI output matches the expected JSON schema with campaign, adset, and ad insights keys  
  - *Config:* JSON schema example with three insight strings  
  - *Input:* AI raw response from OpenAI Chat Model  
  - *Output:* Parsed JSON object with structured insights for next steps  
  - *Edge cases:* Parsing errors if AI output deviates from schema, fallback or retries may be needed  

---

#### 1.5 Variables Preparation for Report

**Overview:**  
Merges the aggregated data and AI insights into a single JSON object to be passed to the PDF generation step.

**Nodes Involved:**  
- Generate variables

**Node Details:**

- **Generate variables**  
  - *Type:* Code  
  - *Role:* Combines the aggregated report JSON with AI insights into one enriched data object  
  - *Config:*  
    - Fetches first JSON from `Format data` node  
    - Injects AI insights fields: `insights_content`, `insights_ad_set_content`, `insights_ads_content` from AI output  
  - *Input:* Output from Generating Insights for Meta Ads (parsed insights) and Format data (aggregated report)  
  - *Output:* Single JSON with full data plus insights for PDF template  
  - *Edge cases:* Missing insights or data inconsistencies could result in incomplete report variables  

---

#### 1.6 PDF Report Generation via pdforge

**Overview:**  
Creates a PDF report using the pdforge service with a pre-defined Meta Ads template and the generated data.

**Nodes Involved:**  
- Pdforge

**Node Details:**

- **Pdforge**  
  - *Type:* pdforge PDF generation node  
  - *Role:* Sends JSON variables to the `meta_ads` template to generate a formatted PDF report  
  - *Config:*  
    - Variables parameter contains stringified JSON of the full report with insights  
    - Template ID is `meta_ads` (pre-made template in pdforge)  
    - Uses pdforge API credentials  
  - *Input:* Output from Generate variables node  
  - *Output:* JSON response with a signed URL to download the generated PDF  
  - *Edge cases:*  
    - Template ID must exist and be correctly configured in pdforge  
    - API key and account must be valid  
    - Network or service downtime could block report generation  

---

#### 1.7 Report Download and Slack Delivery

**Overview:**  
Downloads the generated PDF from pdforge and sends it as a file message to a specified Slack channel with a summary comment.

**Nodes Involved:**  
- Download Binary  
- Slack - Send Message with File

**Node Details:**

- **Download Binary**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the PDF file from the pdforge signed URL  
  - *Config:* URL dynamically extracted from Pdforge node response (`$json.signedUrl`)  
  - *Input:* Pdforge output with signed URL  
  - *Output:* Binary file data (PDF)  
  - *Edge cases:*  
    - Signed URL expiration or invalid URL  
    - Network errors  

- **Slack - Send Message with File**  
  - *Type:* Slack node  
  - *Role:* Uploads the PDF report file to a Slack channel with an introductory comment  
  - *Config:*  
    - File name includes dynamic report date range (e.g., “Meta ADS Report - 2023-07-01 - 2023-07-07”)  
    - Channel ID set to `C093723CAN6` (specific Slack channel)  
    - Initial comment includes emoji and message `"📊 *Here's your Weekly Meta Ads Report* \n"`  
    - Uses OAuth2 Slack credentials  
  - *Input:* Binary PDF file from Download Binary node  
  - *Output:* Slack message confirmation  
  - *Edge cases:*  
    - Slack OAuth token expiration or permission issues  
    - Channel ID must be valid and accessible by the bot  

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                            | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                            |
|--------------------------------|----------------------------------|--------------------------------------------|----------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Weekly at monday 8am            | Schedule Trigger                 | Initiates workflow weekly at Monday 8 a.m. | None                       | Configuration                   |                                                                                                                        |
| Configuration                  | Set                              | Defines Meta Ads account ID and date range | Weekly at monday 8am        | HTTP Request - Meta Ads          | **Here you'll configure:** - AD Account ID - Date range from ads                                                        |
| HTTP Request - Meta Ads        | HTTP Request                    | Retrieves Meta Ads data via Facebook API   | Configuration              | Split Out                      |                                                                                                                        |
| Split Out                     | Split Out                       | Splits array of ads data into individual items | HTTP Request - Meta Ads    | Format data                   |                                                                                                                        |
| Format data                   | Code                            | Aggregates and structures ad performance data | Split Out                  | Generating Insights for Meta Ads |                                                                                                                        |
| Generating Insights for Meta Ads | LangChain Agent                | Generates AI insights from aggregated data | Format data                | Generate variables              |                                                                                                                        |
| OpenAI Chat Model             | LangChain Chat Model             | Provides GPT-4 model for insight generation | Generating Insights for Meta Ads | Structured Output Parser      |                                                                                                                        |
| Structured Output Parser      | LangChain Output Parser          | Parses AI output into structured JSON      | OpenAI Chat Model          | Generating Insights for Meta Ads |                                                                                                                        |
| Generate variables            | Code                            | Combines aggregated data with AI insights | Generating Insights for Meta Ads | Pdforge                      |                                                                                                                        |
| Pdforge                      | pdforge PDF generation           | Creates PDF report from data and template  | Generate variables         | Download Binary                |                                                                                                                        |
| Download Binary              | HTTP Request                    | Downloads PDF file from pdforge signed URL | Pdforge                    | Slack - Send Message with File |                                                                                                                        |
| Slack - Send Message with File | Slack                          | Sends PDF report file with comment to Slack channel | Download Binary            | None                          |                                                                                                                        |
| Sticky Note                  | Sticky Note                     | Introductory explanation and workflow overview | None                       | None                          | ## Welcome to Weekly Meta Ads Report Workflow! This workflow has the following sequence: 1. Time trigger (e.g. every Monday at 8 a.m.) 2. Retrieval of Meta Ads data from the last 7 days (or any period you want) 3. Format it and generate AI insight of the data 4. Transform it into a PDF report using pdforge (Using the pre-made Meta Ads Template) 5. Sending the report as a Slack Message (You can change it to send an e-mail, whatsapp or telegram message) The following accesses are required: - Meta Ads API - pdforge account - AI API access (e.g. OpenAI) - Slack connection. Contact: https://www.linkedin.com/in/marceloamiranda |
| Sticky Note1                 | Sticky Note                     | Notes about parameter configuration        | None                       | None                          | **Here you'll configure:** - AD Account ID - Date range from ads                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure: Weekly interval, trigger day Monday, hour 8 (timezone America/Sao_Paulo)  
   - Name: `Weekly at monday 8am`  

2. **Create Configuration Node**  
   - Type: Set  
   - Set two string fields:  
     - `ad_account_id` = `982086037409625232`  
     - `date_range` = `last_7d`  
   - Connect input from `Weekly at monday 8am`  

3. **Create HTTP Request Node to Fetch Meta Ads Data**  
   - Type: HTTP Request  
   - Name: `HTTP Request - Meta Ads`  
   - Method: GET  
   - URL: `https://graph.facebook.com/v23.0/act_{{$json.ad_account_id}}/insights`  
   - Query Parameters:  
     - `fields`: `campaign_id,campaign_name,adset_id,adset_name,ad_id,ad_name,reach,impressions,inline_link_clicks,inline_link_click_ctr,conversions,spend,cpc,cost_per_conversion,action_values,purchase_roas`  
     - `time_increment`: `1`  
     - `date_preset`: `={{ $json.date_range }}`  
     - `level`: `ad`  
   - Authentication: HTTP Header Auth using Meta Ads API credentials  
   - Enable Pagination by response next URL with interval 500ms  
   - Connect input from `Configuration`  

4. **Add Split Out Node**  
   - Type: Split Out  
   - Field to split: `data`  
   - Connect input from `HTTP Request - Meta Ads`  

5. **Add Code Node to Format and Aggregate Data**  
   - Type: Code  
   - Paste the JavaScript code that:  
     - Parses all ad records  
     - Calculates totals, daily data, top 10 by CPC for campaigns, ad sets, ads  
     - Produces a structured JSON report object  
   - Connect input from `Split Out`  

6. **Add LangChain Agent Node for AI Insights**  
   - Type: LangChain Agent  
   - Configure prompt with system message defining persona and instructions for concise insights in JSON only (see node details)  
   - Input text: JSON stringified data from `Format data` node  
   - Connect input from `Format data`  

7. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1` or latest GPT-4 available  
   - Connect input from `Generating Insights for Meta Ads` agent node's language model input  

8. **Add Structured Output Parser Node**  
   - Type: LangChain Output Parser  
   - Set JSON schema example with three insight keys: `campaign_insights`, `adset_insights`, `ad_insights`  
   - Connect input from `OpenAI Chat Model`  

9. **Connect Structured Output Parser back to `Generating Insights for Meta Ads` node AI output parser input.**  

10. **Add Code Node to Generate Variables for PDF**  
    - Type: Code  
    - Merge the aggregated report JSON (from `Format data`) with AI insights from `Generating Insights for Meta Ads` parsed output  
    - Output single combined JSON for PDF template variables  
    - Connect input from `Generating Insights for Meta Ads`  

11. **Add pdforge Node for PDF Generation**  
    - Type: pdforge PDF Generation node  
    - Template ID: `meta_ads` (pre-created template in pdforge account)  
    - Variables: Pass the JSON stringified combined data from previous step  
    - Connect input from `Generate variables`  
    - Configure pdforge API credentials  

12. **Add HTTP Request Node to Download PDF**  
    - Type: HTTP Request  
    - URL: Use dynamic expression to get signedUrl from pdforge output  
    - Connect input from `Pdforge`  

13. **Add Slack Node to Send PDF File**  
    - Type: Slack  
    - Resource: File  
    - OAuth2 authentication with Slack credentials  
    - Channel ID: `C093723CAN6` (or your channel)  
    - File name: `Meta ADS Report - {{ $('Generate variables').first().json.report_date }}`  
    - Initial comment: `📊 *Here's your Weekly Meta Ads Report* \n`  
    - Connect input from `Download Binary`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                        | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow requires Meta Ads API access configured with HTTP Header Auth credential in n8n.                                                                                                                                                                                          | https://docs.n8n.io/integrations/builtin/credentials/facebookgraph/                                                     |
| Pdforge account is needed for PDF report generation using the pre-made `meta_ads` template.                                                                                                                                                                                        | https://app.pdforge.com/auth/sign-up                                                                                   |
| AI API access required (OpenAI GPT-4 recommended) for generating structured insights.                                                                                                                                                                                              | https://openai.com/api                                                                                                 |
| Slack OAuth2 credentials must have scope for file uploads and messaging.                                                                                                                                                                                                             | https://docs.n8n.io/integrations/builtin/credentials/slack                                                              |
| Contact for questions or support: Marcelo A Miranda on LinkedIn                                                                                                                                                                                                                     | https://www.linkedin.com/in/marceloamiranda                                                                             |
| The workflow’s AI insights agent uses strict prompt constraints to output JSON only with 200-300 character insights per level, ensuring concise, actionable recommendations.                                                                                                        | Internal prompt in `Generating Insights for Meta Ads` node                                                              |
| The workflow is built with fault tolerance for some node failures (e.g., HTTP request errors continue regular output) but may need additional error handling in production for API limits and network issues.                                                                        | Configuration node and HTTP Request node `onError` set to continue                                                      |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.