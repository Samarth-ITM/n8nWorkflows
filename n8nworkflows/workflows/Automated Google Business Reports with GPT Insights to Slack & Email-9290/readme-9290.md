Automated Google Business Reports with GPT Insights to Slack & Email

https://n8nworkflows.xyz/workflows/automated-google-business-reports-with-gpt-insights-to-slack---email-9290


# Automated Google Business Reports with GPT Insights to Slack & Email

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly Google Business performance reports enriched with AI-generated insights. It targets businesses managing multiple Google Business Profiles, enabling them to track key metrics like impressions, clicks, reviews, and sentiment trends over multiple timeframes. The workflow integrates Google Business Profile data, Google Sheets for company and recipient data storage, OpenAI GPT for natural language summarization, Slack for instant messaging, and Gmail for email distribution.

Logical blocks are grouped as follows:

- **1.1 Company and Time Window Initialization**: Loads company data from Google Sheets and builds various date windows (this week, last week, 12-week rolling).
- **1.2 Impressions Data Collection**: Retrieves Google Business Profile metrics (impressions, clicks, requests) over different timeframes and writes them to sheets.
- **1.3 Reviews Data Collection & Summaries**: Gathers and categorizes reviews data across timeframes, calculates sentiment and reply metrics, and writes summaries to sheets.
- **1.4 KPI Aggregation and AI-Powered Summaries**: Joins all metrics, calculates deltas, formats structured data, and generates concise one-liner insights using OpenAI GPT models.
- **1.5 Distribution via Slack and Email**: Merges all AI summaries, attaches recipient info, formats messages, and sends personalized Slack notifications and weekly pulse emails to each company.

---

### 2. Block-by-Block Analysis

#### 1.1 Company and Time Window Initialization

**Overview:**  
This block initializes the workflow by loading companies from a Google Sheet and creating date ranges for reporting periods: this week, last week, and a rolling 12-week window. These date windows are used to query Google Business Profile API data later.

**Nodes Involved:**  
- Read Companies  
- Build Week Window2  
- Build Week Window (reviews)  
- Build last week window  
- Build 12-Week Window  
- Build 12-week window (reviews)  
- Set Company (total reviews)  
- Set Company (reviews)  
- Set Company last week reviews  
- Set Company last 12 week reviews  

**Node Details:**

- **Read Companies**  
  - Type: Google Sheets  
  - Role: Reads company metadata (company name, Google Business accountId, locationId) from a Google Sheet.  
  - Config: Reads from a specified sheet with company info.  
  - Inputs: Triggered by Schedule Trigger.  
  - Outputs: Provides an array of company JSON objects to downstream nodes.  
  - Failure Modes: Google Sheets API errors, OAuth token expiration.  

- **Build Week Window2**  
  - Type: Code  
  - Role: Creates date window for “this week” (last 7 full days ending yesterday) and “last week” (7 days before that) for impressions queries.  
  - Config: UTC midnight calculations, outputs date parts and ISO strings.  
  - Inputs: Company items from Read Companies.  
  - Outputs: Enriched company items with date window fields.  
  - Failure Modes: Date calculation errors if system clock is misconfigured.

- **Build Week Window (reviews)**  
  - Type: Code  
  - Role: Similar to Build Week Window2 but specifically for reviews data; builds date window for this week and last week.  
  - Inputs/Outputs: Analogous to Build Week Window2.  
  - Failure Modes: Same as above.

- **Build last week window**  
  - Type: Code  
  - Role: Builds date window for last week only (7 full days ending yesterday), used for last week reviews and impressions.  
  - Outputs: Single JSON item with last week start/end dates.  
  - Failure Modes: Date calculation errors.

- **Build 12-Week Window**  
  - Type: Code  
  - Role: Constructs a 12-week rolling window ending yesterday, for long-term trend analysis.  
  - Inputs: Company items.  
  - Outputs: Items enriched with 12-week window date parts.  
  - Failure Modes: None significant.

- **Build 12-week window (reviews)**  
  - Type: Code  
  - Role: Analogous to Build 12-Week Window but tailored for reviews data.  
  - Failure Modes: None significant.

- **Set Company (total reviews)**  
  - Type: Code  
  - Role: Maps companies from the “Read Companies” node, formatting company and location IDs and paths for review queries.  
  - Inputs: Items from Read Companies.  
  - Outputs: Items with companyName, accountId, locationId, and API paths.  
  - Failure Modes: Empty company list.

- **Set Company (reviews)**  
  - Same as above, tailored for reviews data queries.

- **Set Company last week reviews**  
  - Same as Set Company (reviews), but used for last week review queries.

- **Set Company last 12 week reviews**  
  - Same mapping logic for last 12 weeks reviews.

---

#### 1.2 Impressions Data Collection

**Overview:**  
Fetches performance metrics from Google Business Profile API for multiple periods: this week, last week, and last 12 weeks. It queries metrics such as website clicks, call clicks, direction requests, and various impression types. The data is flattened and appended back to Google Sheets for persistence.

**Nodes Involved:**  
- Get Impression data  
- Get Impressions (last week)  
- 12 week impressions  
- Flatten  
- Flatten (last week)  
- Flatten 12-week  
- Add Impressions to Sheet  
- Append row in sheet (last week)  
- Add 12 week Impressions to Sheet  

**Node Details:**

- **Get Impression data**  
  - Type: HTTP Request  
  - Role: Calls Google Business Profile API to fetch this week's metrics by locationId and date range, with OAuth2 authentication.  
  - Config: Dynamic URL constructed with date and metrics query parameters.  
  - Inputs: Items enriched with date windows and locationId.  
  - Outputs: API response JSON with metrics time series.  
  - Failure Modes: API quota limits, invalid auth, network errors.

- **Get Impressions (last week)**  
  - Same as above, but URL uses last week date window parameters.

- **12 week impressions**  
  - Same as above, but for 12-week window.

- **Flatten**  
  - Type: Code  
  - Role: Processes API response arrays to sum daily metric values into totals for each company/location/week.  
  - Output: JSON objects with total counts for clicks, requests, impressions broken down by device and platform.  
  - Failure Modes: Missing data arrays, unexpected response formats.

- **Flatten (last week)** and **Flatten 12-week**  
  - Same logic applied to respective API responses.

- **Add Impressions to Sheet**  
  - Type: Google Sheets  
  - Role: Appends processed impression metrics for this week to a dedicated sheet.  
  - Inputs: Flatten node output.  
  - Failure Modes: Google Sheets write errors.

- **Append row in sheet (last week)**  
  - Appends last week impression data similarly.

- **Add 12 week Impressions to Sheet**  
  - Writes 12-week rollup impression data.

---

#### 1.3 Reviews Data Collection & Summaries

**Overview:**  
This block retrieves all reviews across different timeframes (this week, last week, last 12 weeks, all-time), maps and categorizes them by sentiment, reply status, and latency, and summarizes key KPIs. The results are saved back to Google Sheets and fed into AI summarization.

**Nodes Involved:**  
- Get Many Reviews  
- Get last week reviews  
- Get last 12 week reviews  
- Get all reviews (all-time)  
- Map weekly reviews  
- Map + categorize reviews (last week, single node)  
- map and summarize last 12 weeks reviews  
- All-time Reviews → KPIs (no LLM)  
- Summarize Weekly Reviews  
- Summarize Weekly Reviews (last week)  
- Summarize 12-week reviews (totals + weekly average)  
- Add Reviews to Sheet  
- Add Weekly Summary  
- Add all time Summary  

**Node Details:**

- **Get Many Reviews**  
  - Type: Google Business Profile  
  - Role: Fetches all reviews for each company location for the current week or relevant window.  
  - Inputs: Items with account and location paths.  
  - Outputs: Review lists.  
  - Failure Modes: API rate limits, pagination issues.

- **Get last week reviews** and **Get last 12 week reviews**  
  - Same as above for different time windows.

- **Get all reviews**  
  - Gets all reviews without time filtering for all-time KPIs.

- **Map weekly reviews**  
  - Type: Code  
  - Role: Filters and categorizes reviews by sentiment, reply status, and latency within the current week window.  
  - Outputs structured review items with sentiment labels and timing.  
  - Failure Modes: Missing company index, malformed review data.

- **Map + categorize reviews (last week, single node)**  
  - Same as above but for last week.

- **map and summarize last 12 weeks reviews**  
  - Maps filtered reviews for the 12-week window and aggregates sentiment and reply metrics.

- **All-time Reviews → KPIs (no LLM)**  
  - Aggregates lifetime KPIs per company/location from all reviews without language model involvement.

- **Summarize Weekly Reviews**  
  - Aggregates weekly reviews per company: counts by sentiment, average rating, reply rates, median reply latency.

- **Summarize Weekly Reviews (last week)**  
  - Same as above for last week.

- **Summarize 12-week reviews (totals + weekly average)**  
  - Summarizes 12-week review aggregates with averages.

- **Add Reviews to Sheet**  
  - Appends mapped weekly reviews to Google Sheets.

- **Add Weekly Summary**  
  - Appends weekly review summaries.

- **Add all time Summary**  
  - Appends lifetime review KPIs.

---

#### 1.4 KPI Aggregation and AI-Powered Summaries

**Overview:**  
This block merges all impression and review data, calculates comparative deltas (week-over-week, vs 12-week average), formats the data for AI consumption, and generates three distinct one-liner insights using OpenAI GPT: overall performance, review-focused, and impressions-focused.

**Nodes Involved:**  
- Merge2  
- Join All  
- format  
- OpenAI Chat Model  
- OpenAI Chat Model2  
- OpenAI Chat Model4  
- Overall one-liner  
- Reviews one-liner  
- Impressions one-liner  
- overall parser  
- Review Parser  
- Impression parser  

**Node Details:**

- **Merge2**  
  - Type: Merge  
  - Role: Combines multiple data streams (impressions, reviews, all-time summaries) keyed by company/location.

- **Join All**  
  - Type: Code  
  - Role: Consolidates all metrics and reviews, computes deltas, and normalizes data for downstream processing.  
  - Handles complex logic for comparing current vs last week and 12-week averages.  
  - Failure Modes: Missing or mismatched keys.

- **format**  
  - Type: Code  
  - Role: Prepares final JSON with calculated KPIs, deltas, and generates prompt strings for GPT summarization.  
  - Adds flags for "flat" or "up/down" changes for UI.  
  - Failure Modes: Data inconsistencies.

- **OpenAI Chat Model**, **OpenAI Chat Model2**, **OpenAI Chat Model4**  
  - Type: LangChain OpenAI Chat nodes  
  - Role: Generate natural language one-liner summaries for overall, impressions, and reviews data respectively.  
  - Config: Use GPT-5 model, system messages crafted for marketing analyst role, strict JSON output.  
  - Failure Modes: API quota, invalid prompts, network errors.

- **Overall one-liner**, **Reviews one-liner**, **Impressions one-liner**  
  - Type: LangChain Agent nodes with output parsers  
  - Role: Parse and validate GPT outputs as structured JSON per schema.  
  - Failure Modes: Parsing errors if GPT output does not match schema.

- **overall parser**, **Review Parser**, **Impression parser**  
  - Type: Output parser nodes  
  - Role: Define strict JSON schemas for each type of summary output.  
  - Failure Modes: Schema mismatches.

---

#### 1.5 Distribution via Slack and Email

**Overview:**  
Final one-liner summaries are merged and normalized; recipient details from Google Sheets are attached; messages are grouped per company; and personalized Slack messages and HTML-rich weekly pulse emails are sent to each recipient list.

**Nodes Involved:**  
- Merge One Liners  
- Normalize Keys  
- Read Recipients  
- Normalize recipients  
- Attach recipients  
- Group Three Lines Per Company  
- All-companies Slack  
- Per Company Slack  
- Create Email  
- Per company email  

**Node Details:**

- **Merge One Liners**  
  - Type: Merge  
  - Role: Combines the three one-liner types (overall, impressions, reviews) per company/location.

- **Normalize Keys**  
  - Type: Code  
  - Role: Extracts and normalizes company and location keys, selects best available one-liner for each company.

- **Read Recipients**  
  - Type: Google Sheets  
  - Role: Loads recipient contact details (email addresses, Slack channels/users) from a Google Sheet.

- **Normalize recipients**  
  - Type: Code  
  - Role: Constructs join keys and cleans recipient fields to match company data.

- **Attach recipients**  
  - Type: Merge (combine mode)  
  - Role: Joins metrics with recipient contact information by join_key.

- **Group Three Lines Per Company**  
  - Type: Code  
  - Role: Groups all summaries into one message per company; formats message with HTML for email and plain text for Slack; attaches week period and recipient info.

- **All-companies Slack**  
  - Type: Slack  
  - Role: Sends a summary message to a single Slack user (likely admin/monitor).

- **Per Company Slack**  
  - Type: Slack  
  - Role: Sends individual Slack messages to company-specific channels.

- **Create Email**  
  - Type: Code  
  - Role: Builds the HTML email body for the weekly pulse email combining all metrics and one-liners.

- **Per company email**  
  - Type: Gmail  
  - Role: Sends the weekly pulse email to each company’s recipient list with optional CC.

---

### 3. Summary Table

| Node Name                         | Node Type                               | Functional Role                                   | Input Node(s)                                 | Output Node(s)                              | Sticky Note                                                                                                                                                                                                                                                                                                                                                               |
|----------------------------------|---------------------------------------|--------------------------------------------------|-----------------------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger                      | Starts workflow weekly                           |                                               | Read Companies                              | # 🧩 Setup Guide: Google Business → Slack + Weekly Pulse Email Workflow ... (full setup instructions and prerequisites with links)                                                                                                                                                                                                                                        |
| Read Companies                  | Google Sheets                        | Reads company list                               | Schedule Trigger                              | Build Week Window2, Build Week Window (reviews), Build 12-Week Window, Set Company (total reviews), Build last week window, Build 12-week window (reviews) | Gather company data from google sheets                                                                                                                                                                                                                                                                                                                                  |
| Build Week Window2              | Code                                | Builds this week & last week date window        | Read Companies                                | Get Impression data, Get Impressions (last week) | Impressions Data Collection - This section gathers visibility metrics...                                                                                                                                                                                                                                                                                                  |
| Build Week Window (reviews)     | Code                                | Builds this week & last week window for reviews | Read Companies                                | Set Company (reviews)                       | Reviews Data Collection & Summaries - This section gathers review data...                                                                                                                                                                                                                                                                                                 |
| Build last week window          | Code                                | Builds last week window (7 days)                  | Read Companies                                | Set Company last week reviews               | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Build 12-Week Window            | Code                                | Builds rolling 12-week window                    | Read Companies                                | 12 week impressions                        | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Build 12-week window (reviews)  | Code                                | Builds rolling 12-week window for reviews        | Read Companies                                | Set Company last 12 week reviews            | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Set Company (total reviews)     | Code                                | Formats company info for all-time reviews         | Read Companies                                | Get all reviews                            | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Set Company (reviews)           | Code                                | Formats company info for weekly reviews           | Read Companies                                | Get Many Reviews                           | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Set Company last week reviews   | Code                                | Formats company info for last week reviews        | Build last week window                        | Get last week reviews                      | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Set Company last 12 week reviews| Code                                | Formats company info for last 12 week reviews     | Build 12-week window (reviews)                | Get last 12 week reviews                   | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Get Impression data             | HTTP Request                        | Fetches this week Google Business metrics         | Build Week Window2                            | Flatten                                    | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Get Impressions (last week)     | HTTP Request                        | Fetches last week Google Business metrics         | Build Week Window2                            | Flatten (last week)                        | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| 12 week impressions            | HTTP Request                        | Fetches 12-week rolling Google Business metrics   | Build 12-Week Window                          | Flatten 12-week                            | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Flatten                        | Code                                | Aggregates API response metrics into totals       | Get Impression data                           | Add Impressions to Sheet                   | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Flatten (last week)             | Code                                | Aggregates last week metrics                       | Get Impressions (last week)                   | Append row in sheet                        | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Flatten 12-week                | Code                                | Aggregates 12-week metrics                         | 12 week impressions                          | Add 12 week Impressions to Sheet           | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Add Impressions to Sheet        | Google Sheets                      | Stores this week impression data                   | Flatten                                       | Merge2                                     | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Append row in sheet             | Google Sheets                      | Stores last week impression data                   | Flatten (last week)                           | Merge2                                     | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Add 12 week Impressions to Sheet| Google Sheets                      | Stores 12-week impression data                     | Flatten 12-week                              | Merge2                                     | Impressions Data Collection                                                                                                                                                                                                                                                                                                                                               |
| Get Many Reviews               | Google Business Profile             | Fetches all reviews for current week               | Set Company (reviews)                         | Map weekly reviews                        | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Get last week reviews          | Google Business Profile             | Fetches all reviews for last week                   | Set Company last week reviews                 | Map + categorize reviews (last week, single node) | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Get last 12 week reviews       | Google Business Profile             | Fetches all reviews for last 12 weeks               | Set Company last 12 week reviews              | map and summarize last 12 weeks reviews   | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Get all reviews                | Google Business Profile             | Fetches all historical reviews                      | Set Company (total reviews)                   | All-time Reviews → KPIs (no LLM)          | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Map weekly reviews             | Code                              | Filters and categorizes current week reviews       | Get Many Reviews                              | Add Reviews to Sheet                       | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Map + categorize reviews (last week, single node)| Code                   | Filters and categorizes last week reviews          | Get last week reviews                         | Summarize Weekly Reviews (last week)      | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| map and summarize last 12 weeks reviews| Code                      | Maps and summarizes 12-week reviews                 | Get last 12 week reviews                      | Summarize 12-week reviews (totals + weekly average) | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| All-time Reviews → KPIs (no LLM)| Code                            | Aggregates lifetime KPIs from all reviews           | Get all reviews                              | Add all time Summary                      | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Summarize Weekly Reviews       | Code                              | Summarizes weekly review metrics                     | Add Reviews to Sheet                          | Add Weekly Summary                        | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Summarize Weekly Reviews (last week)| Code                        | Summarizes last week review metrics                  | Map + categorize reviews (last week, single node) | Add Weekly Summary                       | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Summarize 12-week reviews (totals + weekly average)| Code                   | Summarizes 12-week review metrics                    | map and summarize last 12 weeks reviews      | Merge2                                     | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Add Reviews to Sheet           | Google Sheets                    | Stores mapped weekly reviews                          | Map weekly reviews                            | Summarize Weekly Reviews                  | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Add Weekly Summary             | Google Sheets                    | Stores summarized weekly review KPI                   | Summarize Weekly Reviews                      | Merge2                                     | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Add all time Summary           | Google Sheets                    | Stores all-time review KPIs                            | All-time Reviews → KPIs (no LLM)              | Merge2                                     | Reviews Data Collection & Summaries                                                                                                                                                                                                                                                                                                                                      |
| Merge2                        | Merge                             | Combines all data streams (impressions, reviews)     | Add Impressions to Sheet, Append row in sheet, Add 12 week Impressions to Sheet, Add Weekly Summary, Summarize Weekly Reviews (last week), Summarize 12-week reviews, Add all time Summary | Join All                                    | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Join All                      | Code                              | Aggregates and compares multi-source KPI data        | Merge2                                       | format                                     | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| format                       | Code                              | Calculates deltas, formats KPI data, constructs GPT prompts | Join All                                     | OpenAI Chat Model, OpenAI Chat Model2, OpenAI Chat Model4 | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| OpenAI Chat Model             | LangChain OpenAI Chat             | Generates overall performance one-liner summary     | format                                        | Overall one-liner                         | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| OpenAI Chat Model2            | LangChain OpenAI Chat             | Generates impressions one-liner summary              | format                                        | Impressions one-liner                    | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| OpenAI Chat Model4            | LangChain OpenAI Chat             | Generates reviews one-liner summary                   | format                                        | Reviews one-liner                        | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Overall one-liner             | LangChain Agent                   | Parses GPT output for overall summary                 | OpenAI Chat Model                             | Merge One Liners                         | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Reviews one-liner             | LangChain Agent                   | Parses GPT output for reviews summary                  | OpenAI Chat Model4                            | Merge One Liners                         | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Impressions one-liner         | LangChain Agent                   | Parses GPT output for impressions summary             | OpenAI Chat Model2                            | Merge One Liners                         | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| overall parser                | Output Parser                    | Validates overall one-liner JSON output                | OpenAI Chat Model                             | Overall one-liner                       | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Review Parser                | Output Parser                    | Validates reviews one-liner JSON output                | OpenAI Chat Model4                            | Reviews one-liner                       | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Impression parser             | Output Parser                    | Validates impressions one-liner JSON output           | OpenAI Chat Model2                            | Impressions one-liner                   | LLM Summaries & One-Liners                                                                                                                                                                                                                                                                                                                                                 |
| Merge One Liners             | Merge                             | Combines all three one-liners per company              | Overall one-liner, Reviews one-liner, Impressions one-liner | Normalize Keys, Read Recipients          | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Normalize Keys               | Code                              | Normalizes keys and picks best one-liner               | Merge One Liners                              | Attach recipients                       | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Read Recipients              | Google Sheets                    | Reads email and Slack recipient data                    | Normalize Keys                               | Normalize recipients                    | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Normalize recipients         | Code                              | Constructs join keys and normalizes recipient fields   | Read Recipients                              | Attach recipients                       | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Attach recipients            | Merge (combine)                   | Joins metrics with recipient info                        | Normalize recipients, Normalize Keys         | Group Three Lines Per Company            | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Group Three Lines Per Company | Code                              | Groups and formats all summaries and recipient info    | Attach recipients                            | All-companies Slack, Per Company Slack, Create Email | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| All-companies Slack          | Slack                            | Sends combined summary to admin Slack user             | Group Three Lines Per Company                  |                                             | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Per Company Slack            | Slack                            | Sends individual Slack messages to company channels    | Group Three Lines Per Company                  |                                             | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Create Email                 | Code                              | Builds HTML email body with formatted metrics          | Group Three Lines Per Company                  | Per company email                       | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |
| Per company email            | Gmail                            | Sends weekly pulse email to company's recipients       | Create Email                                  |                                             | Distribution: Slack & Weekly Pulse Emails                                                                                                                                                                                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger weekly at your preferred hour (e.g., every week, 18:00).

2. **Add a Google Sheets node "Read Companies"**  
   - Connect credentials for Google Sheets OAuth2.  
   - Configure to read your companies sheet with columns: company, accountId, locationId, slack_channel, email recipients, etc.

3. **Add Code nodes to build date windows:**  
   - "Build Week Window2": Calculate this week and last week date parts for impressions.  
   - "Build Week Window (reviews)": Similar window for reviews data.  
   - "Build last week window": Date window for last week only.  
   - "Build 12-Week Window": 12-week rolling window for impressions.  
   - "Build 12-week window (reviews)": 12-week rolling window for reviews.

4. **Add Code nodes to set company data (paths) for API calls:**  
   - "Set Company (total reviews)"  
   - "Set Company (reviews)"  
   - "Set Company last week reviews"  
   - "Set Company last 12 week reviews"  
   - Each maps company/location info from "Read Companies" to the API path format.

5. **Add HTTP Request nodes to fetch impressions data:**  
   - "Get Impression data" for this week.  
   - "Get Impressions (last week)" for last week.  
   - "12 week impressions" for 12-week window.  
   - Configure with Google Business Profile OAuth2 credentials and dynamic URLs using date parts.

6. **Add Code nodes to flatten impressions data:**  
   - "Flatten" for this week data.  
   - "Flatten (last week)" for last week data.  
   - "Flatten 12-week" for 12-week data.

7. **Add Google Sheets nodes to append impressions data:**  
   - "Add Impressions to Sheet" (this week).  
   - "Append row in sheet" (last week).  
   - "Add 12 week Impressions to Sheet" (12 weeks).

8. **Add Google Business Profile nodes to fetch reviews:**  
   - "Get Many Reviews" (all current week).  
   - "Get last week reviews".  
   - "Get last 12 week reviews".  
   - "Get all reviews" (all-time).

9. **Add Code nodes to map and categorize reviews:**  
   - "Map weekly reviews" for current week.  
   - "Map + categorize reviews (last week, single node)".  
   - "map and summarize last 12 weeks reviews".  
   - "All-time Reviews → KPIs (no LLM)".

10. **Add Code nodes to summarize reviews:**  
    - "Summarize Weekly Reviews".  
    - "Summarize Weekly Reviews (last week)".  
    - "Summarize 12-week reviews (totals + weekly average)".

11. **Add Google Sheets nodes to append reviews data and summaries:**  
    - "Add Reviews to Sheet".  
    - "Add Weekly Summary".  
    - "Add all time Summary".

12. **Add Merge node "Merge2"**  
    - Inputs: All impression and review summary nodes.  
    - Configure to merge by company/location keys.

13. **Add Code node "Join All"**  
    - Combine all KPI data, calculate deltas, and normalize.

14. **Add Code node "format"**  
    - Calculate final KPI deltas and build GPT prompt strings.

15. **Add three OpenAI Chat Model nodes:**  
    - "OpenAI Chat Model" for overall one-liner (GPT-5).  
    - "OpenAI Chat Model2" for impressions one-liner.  
    - "OpenAI Chat Model4" for reviews one-liner.  
    - Provide OpenAI API credentials.

16. **Add corresponding LangChain Agent nodes:**  
    - "Overall one-liner" parses overall summary.  
    - "Impressions one-liner" parses impressions summary.  
    - "Reviews one-liner" parses reviews summary.

17. **Add Output Parser nodes:**  
    - "overall parser", "Impression parser", and "Review Parser" with appropriate JSON schemas.

18. **Add Merge node "Merge One Liners"**  
    - Combine the three one-liners per company/location.

19. **Add Code node "Normalize Keys"**  
    - Normalize keys and select best one-liner per company.

20. **Add Google Sheets node "Read Recipients"**  
    - Load Slack and email recipient info.

21. **Add Code node "Normalize recipients"**  
    - Build join keys and clean recipient data.

22. **Add Merge node "Attach recipients"**  
    - Combine metrics with recipients using join_key.

23. **Add Code node "Group Three Lines Per Company"**  
    - Group all one-liners with recipient info and week period.  
    - Format message text and HTML.

24. **Add Slack nodes:**  
    - "All-companies Slack" to send admin summary.  
    - "Per Company Slack" to send company-specific Slack messages.

25. **Add Code node "Create Email"**  
    - Build HTML email with KPI data and one-liners.

26. **Add Gmail node "Per company email"**  
    - Send the weekly pulse email to company recipients.

27. **Connect nodes according to described workflow connections.**

28. **Configure all credentials:**  
    - Google Sheets OAuth2  
    - Google Business Profile OAuth2  
    - OpenAI API  
    - Slack OAuth2  
    - Gmail OAuth2 or SMTP

29. **Test and validate each block independently before activating the full workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                                                                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🧩 Setup Guide: Google Business → Slack + Weekly Pulse Email Workflow. Prerequisites include Google Sheets template, Google Business API OAuth2 setup, Google Sheets OAuth2, Slack OAuth2, and optional Gmail/SMTP credential. Template: https://docs.google.com/spreadsheets/d/1u-87I9CUywZwiy9oDTUkMrVSADzeojk2UTTJRs3YOGY/edit?usp=sharing                                                                                                                                                                   | Sticky Note at workflow start; comprehensive setup instructions.                                                                                                                                                                                          |
| Impressions Data Collection: Pulls visibility metrics from Google Business Profiles for multiple periods and writes them to Google Sheets for later use.                                                                                                                                                                                                                                                                                                                                              | Sticky Note near impressions data nodes.                                                                                                                                                                                                                     |
| Reviews Data Collection & Summaries: Gathers, maps, categorizes, and summarizes reviews data with sentiment and reply analytics for weekly, last week, 12-week, and all-time periods.                                                                                                                                                                                                                                                                                                                        | Sticky Note near reviews data nodes.                                                                                                                                                                                                                        |
| LLM Summaries & One-Liners: Combines structured KPIs and generates succinct natural-language insights using OpenAI GPT-5, feeding Slack and email outputs.                                                                                                                                                                                                                                                                                                                                           | Sticky Note near AI summary nodes.                                                                                                                                                                                                                          |
| Distribution: Slack & Weekly Pulse Emails: Final step sends AI summaries to Slack channels and email recipients per company with formatted messages.                                                                                                                                                                                                                                                                                                                                                   | Sticky Note near distribution nodes.                                                                                                                                                                                                                        |
| The workflow is optimized for UTC-based date calculations to ensure consistent weekly reporting across time zones.                                                                                                                                                                                                                                                                                                                                                                                     | General operational note inferred from code nodes.                                                                                                                                                                                                          |
| The OpenAI nodes use GPT-5 model with strict JSON output enforced via LangChain output parsers to maintain data integrity.                                                                                                                                                                                                                                                                                                                                                                             | Best practice for reliable AI integration.                                                                                                                                                                                                                   |
| Slack OAuth2 credential requires permissions: chat:write, users:read, channels:read.                                                                                                                                                                                                                                                                                                                                                                                                                     | Credential setup note from sticky note.                                                                                                                                                                                                                      |
| Google Business Profile API calls may be rate-limited; consider adding retry or throttling logic if needed.                                                                                                                                                                                                                                                                                                                                                                                            | Potential failure mode suggestion.                                                                                                                                                                                                                           |
| Email sending uses Gmail OAuth2 credential but can be adapted to SMTP for other providers.                                                                                                                                                                                                                                                                                                                                                                                                               | Flexibility note for email node.                                                                                                                                                                                                                            |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal or offensive material. All data handled is legally permissible and public.