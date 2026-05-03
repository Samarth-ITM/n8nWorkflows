Monitor NASA asteroid threats with AI fact-check and multi-channel alerts

https://n8nworkflows.xyz/workflows/monitor-nasa-asteroid-threats-with-ai-fact-check-and-multi-channel-alerts-11289


# Monitor NASA asteroid threats with AI fact-check and multi-channel alerts

### 1. Workflow Overview

This workflow, titled **Real-time Asteroid Alert System with AI Fact-Check and Multi-Channel Notifications**, is designed to monitor potential asteroid threats using NASA's Near Earth Object (NEO) data. It performs a 7-day forecast scan of asteroids, identifies hazardous objects, and enriches this data by searching news coverage across three international regions (US, Japan, EU/UK). Using AI-powered fact-checking, it compares media reports against NASA data to detect misinformation and sensationalism. The results are scored for actual threat and media panic levels, visualized through charts, and distributed via multiple communication channels such as Slack, Discord, Email, and optionally logged into Google Sheets.

The workflow logically divides into these key blocks:

- **1.1 Input Reception & Triggering**: Accepts scheduled daily triggers and webhook calls to start the process.
- **1.2 NASA Asteroid Data Acquisition & Analysis**: Fetches and analyzes asteroid data for threat assessment.
- **1.3 Regional News Search Setup and Execution**: Configures and performs news searches for relevant asteroid news in multiple languages and regions.
- **1.4 AI Fact-Checking & Analysis**: Uses OpenAI to fact-check and score news reports against NASA data.
- **1.5 Visualization Generation**: Creates graphical charts representing threat and media panic scores and regional tone comparisons.
- **1.6 Multi-Channel Notification Formatting and Dispatch**: Formats messages tailored for Slack, Discord, and Email, sending alerts and responding to webhook calls.
- **1.7 Logging and Finalization**: Optionally logs alert data to Google Sheets for historical tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Triggering

**Overview:**  
This block initiates the workflow either on a scheduled daily basis (9 AM) or via an HTTP POST webhook, allowing both automated and manual triggering.

**Nodes Involved:**  
- Daily Schedule  
- Webhook Trigger  
- Merge Triggers

**Node Details:**

- **Daily Schedule**  
  - *Type:* Schedule Trigger  
  - *Role:* Fires workflow every day at 9 AM local time.  
  - *Config:* Trigger hour set to 9.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connected to Merge Triggers node.  
  - *Edge Cases:* If n8n instance time zone changes, schedule timing may shift.  
  - *Version:* 1.2

- **Webhook Trigger**  
  - *Type:* Webhook  
  - *Role:* Accepts external POST requests at `/webhook/asteroid-alert`.  
  - *Config:* POST method; responseMode set to responseNode to send a response later.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connected to Merge Triggers node.  
  - *Edge Cases:* Can fail if webhook URL is incorrect or request format is invalid.  
  - *Version:* 2

- **Merge Triggers**  
  - *Type:* Merge  
  - *Role:* Merges two trigger sources (schedule and webhook) into one flow.  
  - *Config:* Defaults (no specific parameters).  
  - *Inputs:* From Daily Schedule and Webhook Trigger nodes.  
  - *Outputs:* Connected to NASA data retrieval.  
  - *Edge Cases:* If both triggers fire simultaneously, both inputs are merged and processed sequentially.  
  - *Version:* 3

---

#### 1.2 NASA Asteroid Data Acquisition & Analysis

**Overview:**  
Retrieves 7 days of asteroid data from NASA’s NeoWs API and analyzes it to determine threat levels and identify the largest and hazardous asteroids.

**Nodes Involved:**  
- Get NASA Asteroid Data  
- Analyze Asteroid Threats (Code)  
- Check for Hazardous Objects (If)  
- Create All-Clear Status (Code)

**Node Details:**

- **Get NASA Asteroid Data**  
  - *Type:* NASA node (API integration)  
  - *Role:* Fetches Near Earth Object feed data from NASA for today plus 7 days ahead.  
  - *Config:* Start date = today, End date = today + 7 days (dynamic expressions).  
  - *Inputs:* From Merge Triggers.  
  - *Outputs:* To Analyze Asteroid Threats.  
  - *Edge Cases:* Possible failures include API key rate limits, invalid credentials, or network issues.  
  - *Version:* 1

- **Analyze Asteroid Threats**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Processes raw asteroid data to:  
    - Identify hazardous asteroids  
    - Find the biggest asteroid  
    - Determine alert level (LOW, MEDIUM, HIGH) based on hazard count and proximity (< 10 lunar distances)  
  - *Config:* Custom JS iterating over each asteroid, extracting key metrics like size, velocity, miss distance, etc.  
  - *Key Expressions:* Uses `$input.all()` to process all asteroid data items.  
  - *Inputs:* NASA data items.  
  - *Outputs:* Structured JSON with analysis results.  
  - *Edge Cases:* Missing or malformed data could cause undefined fields; defensive programming used to default zero values.  
  - *Version:* 2

- **Check for Hazardous Objects**  
  - *Type:* If node  
  - *Role:* Branches workflow based on whether any hazardous asteroids are detected (`hazardous_count > 0`).  
  - *Config:* Numeric greater-than check on `hazardous_count`.  
  - *Inputs:* From Analyze Asteroid Threats.  
  - *Outputs:*  
    - True: proceeds to Configure Regional Search.  
    - False: proceeds to Create All-Clear Status.  
  - *Edge Cases:* If data field missing or zero, false branch executes safely.  
  - *Version:* 2

- **Create All-Clear Status**  
  - *Type:* Code node  
  - *Role:* Generates a simple ALL_CLEAR status message when no hazardous objects are found.  
  - *Config:* Returns JSON with status, message, total asteroids scanned, biggest asteroid info, and scan date.  
  - *Inputs:* From false branch of Check for Hazardous Objects.  
  - *Outputs:* Not connected further in this workflow (terminal for no hazard scenario).  
  - *Version:* 2

---

#### 1.3 Regional News Search Setup and Execution

**Overview:**  
Prepares search queries for news coverage in three regions (US, Japan, EU/UK) and runs them through Apify’s Google Search scraper to retrieve recent asteroid-related news.

**Nodes Involved:**  
- Configure Regional Search (Code)  
- Loop Over Regions (SplitInBatches)  
- Search Regional News (Apify)  
- Aggregate All News

**Node Details:**

- **Configure Regional Search**  
  - *Type:* Code node  
  - *Role:* Generates three search query objects with language, country, query suffix, and asteroid name for US (English), Japan (Japanese), and EU/UK (English).  
  - *Config:* Uses hazardous asteroid or biggest asteroid name as search target.  
  - *Outputs:* Array of three JSON objects for batch processing.  
  - *Version:* 2

- **Loop Over Regions**  
  - *Type:* SplitInBatches node  
  - *Role:* Iterates over the three region query objects individually to process each search separately.  
  - *Config:* Default batch size (1).  
  - *Inputs:* From Configure Regional Search or Search Regional News (looping).  
  - *Outputs:*  
    - Main output to Aggregate All News (collect results).  
    - Secondary output back to Search Regional News for recursive scraping.  
  - *Edge Cases:* May loop indefinitely if misconfigured recursion; here carefully designed to continue only once per region.  
  - *Version:* 3

- **Search Regional News**  
  - *Type:* Apify node (actor execution)  
  - *Role:* Runs Apify’s Google Search scraper actor to retrieve news results for the current region’s query.  
  - *Config:*  
    - Actor: Google Search Scraper (predefined ID).  
    - Queries built dynamically from `search_name` + `query_suffix`.  
    - Max 1 page per query, 5 results per page.  
    - Uses OAuth2 credential for Apify.  
  - *Inputs:* From Loop Over Regions.  
  - *Outputs:* Loops back to Loop Over Regions to process next batch.  
  - *Edge Cases:* Possible failures include OAuth token expiration, API quota limits, or actor errors.  
  - *Version:* 1

- **Aggregate All News**  
  - *Type:* Aggregate node  
  - *Role:* Combines all news results from the batches into a single array under `news_results` field.  
  - *Inputs:* From Loop Over Regions (main output).  
  - *Outputs:* To AI Fact-Check Analysis.  
  - *Version:* 1

---

#### 1.4 AI Fact-Checking & Analysis

**Overview:**  
Uses OpenAI GPT-4o-mini model to fact-check news articles against NASA data, scoring threat levels, media sensationalism, misinformation detection, and regional tone analysis.

**Nodes Involved:**  
- AI Fact-Check Analysis (OpenAI)  
- Parse AI Response (Code)

**Node Details:**

- **AI Fact-Check Analysis**  
  - *Type:* OpenAI (LangChain) node  
  - *Role:* Sends aggregated news and asteroid data to GPT-4o-mini model for deep AI analysis.  
  - *Config:* Model set to "gpt-4o-mini" without special prompt overrides (assumed integrated prompt in node).  
  - *Inputs:* Aggregated news data.  
  - *Outputs:* Raw AI response JSON.  
  - *Edge Cases:* API key limits, network timeouts, or malformed input can cause failures.  
  - *Version:* 2

- **Parse AI Response**  
  - *Type:* Code node  
  - *Role:* Extracts and parses JSON from the AI response text, handling cases where the response is wrapped in markdown code blocks.  
  - *Config:* Uses regex to find JSON inside triple backticks, tries JSON.parse, and falls back to default error response on failure.  
  - *Inputs:* AI Fact-Check Analysis output.  
  - *Outputs:* Parsed structured JSON with threat scores, misinformation flags, regional tone, and summary.  
  - *Edge Cases:* Unexpected AI response format or parse errors handled gracefully with fallback JSON.  
  - *Version:* 2

---

#### 1.5 Visualization Generation

**Overview:**  
Generates URLs for graphical charts representing threat level, media panic, and regional media tone comparison using QuickChart.io.

**Nodes Involved:**  
- Generate Visualization Charts (Code)

**Node Details:**

- **Generate Visualization Charts**  
  - *Type:* Code node  
  - *Role:* Builds three chart configurations:  
    - Threat level gauge (green-yellow-red zones)  
    - Media panic gauge  
    - Bar chart comparing sensationalism levels across US, Japan, and EU  
  - *Config:* Converts tones to numeric scores; encodes charts as URLs via QuickChart.io.  
  - *Inputs:* Parsed AI analysis and asteroid data.  
  - *Outputs:* JSON with chart URLs and scores.  
  - *Edge Cases:* Missing data defaults scores to zero; URL encoding errors unlikely but possible.  
  - *Version:* 2

---

#### 1.6 Multi-Channel Notification Formatting and Dispatch

**Overview:**  
Formats detailed alert messages for Slack, Discord, and Email with threat data, AI analysis, and charts; sends notifications and responds to webhook calls.

**Nodes Involved:**  
- Format Multi-Channel Messages (Code)  
- Send Slack Alert  
- Send Discord Alert  
- Send Email Alert  
- Webhook Response

**Node Details:**

- **Format Multi-Channel Messages**  
  - *Type:* Code node  
  - *Role:* Builds formatted messages:  
    - Slack: Markdown with emoji, bullet points, fact-check results, and recommendation.  
    - Discord: Adapted from Slack with bold markdown.  
    - Email: Full HTML email with inline charts and structured sections, including misinformation alert if detected.  
  - *Inputs:* Visualization charts and analysis data.  
  - *Outputs:* JSON with formatted messages for each channel.  
  - *Edge Cases:* Null or missing fields handled with default fallback texts.  
  - *Version:* 2

- **Send Slack Alert**  
  - *Type:* Slack node  
  - *Role:* Sends alert to a configured Slack channel with a rich attachment including color-coded alert and chart image.  
  - *Config:* OAuth2 authentication, channel ID specified via credential or parameter.  
  - *Inputs:* Formatted Slack message and alert color.  
  - *Edge Cases:* OAuth token expiry, channel access errors, rate limits.  
  - *Version:* 2.3

- **Send Discord Alert**  
  - *Type:* Discord node  
  - *Role:* Posts alert message to Discord via webhook URL with embedded formatting.  
  - *Config:* Webhook authentication; message content is Discord-formatted string.  
  - *Edge Cases:* Invalid webhook URL, network issues.  
  - *Version:* 2

- **Send Email Alert**  
  - *Type:* Email Send node  
  - *Role:* Sends HTML email alert to recipients with subject, body, and charts.  
  - *Config:* SMTP or other mail credentials configured; from and to emails specified.  
  - *Edge Cases:* SMTP failure, invalid recipient addresses.  
  - *Version:* 2.1

- **Webhook Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends JSON response back to the webhook POST caller confirming success and summary data (alert level, hazardous count, threat score).  
  - *Config:* Responds with JSON containing success status and key results.  
  - *Edge Cases:* If previous nodes fail, response may not be sent or contain error info.  
  - *Version:* 1.1

---

#### 1.7 Logging and Finalization

**Overview:**  
Optionally logs alert summaries and scores into a Google Sheets document for historical tracking.

**Nodes Involved:**  
- Log to Google Sheets

**Node Details:**

- **Log to Google Sheets**  
  - *Type:* Google Sheets node  
  - *Role:* Appends a new row with alert data including date, alert level, threat score, media panic score, biggest asteroid, misinformation flag, and region info.  
  - *Config:* Document ID and Sheet name specified; columns mapped explicitly with expressions drawing from analysis nodes.  
  - *Inputs:* From Format Multi-Channel Messages (final).  
  - *Edge Cases:* API quota limits, permission errors, invalid spreadsheet ID or sheet name.  
  - *Version:* 4.5

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                          | Input Node(s)                  | Output Node(s)                                 | Sticky Note                                                                                                  |
|---------------------------|-------------------------|----------------------------------------|-------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Daily Schedule            | Schedule Trigger        | Daily 9 AM trigger                     | None                          | Merge Triggers                                | ## 🔄 Triggers<br>**Schedule**: Daily at 9 AM<br>**Webhook**: POST `/webhook/asteroid-alert` Both enabled.   |
| Webhook Trigger           | Webhook                 | External webhook trigger                | None                          | Merge Triggers                                | See above                                                                                                    |
| Merge Triggers            | Merge                   | Merges schedule and webhook triggers   | Daily Schedule, Webhook Trigger| Get NASA Asteroid Data                        | See above                                                                                                    |
| Get NASA Asteroid Data     | NASA API                | Retrieves 7-day asteroid data          | Merge Triggers                | Analyze Asteroid Threats                       | ## ⚙️ Setup Instructions<br>Step 1: NASA API key required.                                                   |
| Analyze Asteroid Threats   | Code                    | Analyzes hazard, biggest asteroid, alert level | Get NASA Asteroid Data        | Check for Hazardous Objects                    | ## 🔍 Asteroid Analysis<br>Scans 7 days data; defines alert levels LOW, MEDIUM, HIGH.                        |
| Check for Hazardous Objects| If                      | Branch based on hazardous count        | Analyze Asteroid Threats       | Configure Regional Search (true), Create All-Clear Status (false) | See above                                                                                                    |
| Create All-Clear Status    | Code                    | Generates all-clear message             | Check for Hazardous Objects (false) | None (terminal)                              | See above                                                                                                    |
| Configure Regional Search  | Code                    | Prepares regional news search queries  | Check for Hazardous Objects (true) | Loop Over Regions                            | ## 🌍 News Search<br>Searches US (English), Japan (Japanese), EU/UK (English), 5 results each.                |
| Loop Over Regions          | SplitInBatches          | Iterates over regional search queries  | Configure Regional Search, Search Regional News | Aggregate All News (main), Search Regional News (recursive) | See above                                                                                                    |
| Search Regional News       | Apify                   | Scrapes news results via Google Search | Loop Over Regions             | Loop Over Regions                             | ## ⚙️ Setup Instructions<br>Step 2: Apify OAuth credentials required.                                        |
| Aggregate All News         | Aggregate               | Combines news results into single array| Loop Over Regions             | AI Fact-Check Analysis                        | See above                                                                                                    |
| AI Fact-Check Analysis     | OpenAI (LangChain)      | AI fact-check and scoring               | Aggregate All News            | Parse AI Response                             | ## 🤖 AI Fact-Check<br>Verifies news vs NASA data; scores threat, panic, tone, misinformation.               |
| Parse AI Response          | Code                    | Parses AI JSON response, handles errors| AI Fact-Check Analysis        | Generate Visualization Charts                 | See above                                                                                                    |
| Generate Visualization Charts| Code                  | Creates chart URLs for threat and panic| Parse AI Response             | Format Multi-Channel Messages                 | See above                                                                                                    |
| Format Multi-Channel Messages| Code                   | Formats alert messages for Slack, Discord, Email | Generate Visualization Charts | Send Slack Alert, Send Discord Alert, Send Email Alert, Webhook Response, Log to Google Sheets | ## 📤 Output Channels<br>Slack, Discord, Email, Webhook, Google Sheets optional.                            |
| Send Slack Alert           | Slack                   | Sends Slack alert with rich attachment | Format Multi-Channel Messages | None                                          | See above                                                                                                    |
| Send Discord Alert         | Discord                 | Sends Discord alert via webhook         | Format Multi-Channel Messages | None                                          | See above                                                                                                    |
| Send Email Alert           | Email Send              | Sends HTML email alert                   | Format Multi-Channel Messages | None                                          | See above                                                                                                    |
| Webhook Response           | Respond to Webhook      | Responds to webhook caller with success | Format Multi-Channel Messages | None                                          | See above                                                                                                    |
| Log to Google Sheets       | Google Sheets           | Appends alert log entry                  | Format Multi-Channel Messages | None                                          | ## ⚙️ Setup Instructions<br>Step 5: Optional Google Sheets logging requires Sheet setup and credentials.    |
| Sticky Note1               | Sticky Note             | Setup instructions                       | None                         | None                                          | See content in Workflow Overview                                                                             |
| Sticky Note2               | Sticky Note             | Trigger explanations                     | None                         | None                                          | See above                                                                                                    |
| Sticky Note3               | Sticky Note             | Asteroid analysis summary                | None                         | None                                          | See above                                                                                                    |
| Sticky Note4               | Sticky Note             | News search regions and details          | None                         | None                                          | See above                                                                                                    |
| Sticky Note5               | Sticky Note             | AI fact-check summary                     | None                         | None                                          | See above                                                                                                    |
| Sticky Note6               | Sticky Note             | Output channels instructions              | None                         | None                                          | See above                                                                                                    |
| Sticky Note7               | Sticky Note             | Overall workflow description and target users | None                   | None                                          | See above                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Daily Schedule" node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9 AM.  
   - No credentials needed.

2. **Create the "Webhook Trigger" node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `asteroid-alert`  
   - Response Mode: Response Node  
   - No credentials; used for external triggering.

3. **Create "Merge Triggers" node:**  
   - Type: Merge  
   - Connect outputs of Daily Schedule and Webhook Trigger to inputs of this node.

4. **Create "Get NASA Asteroid Data" node:**  
   - Type: NASA  
   - Resource: asteroidNeoFeed  
   - Parameters:  
     - Start Date: `={{ $today.format('YYYY-MM-DD') }}`  
     - End Date: `={{ $today.plus(7, 'days').format('YYYY-MM-DD') }}`  
   - Add NASA API credentials (must be set up beforehand).

5. **Create "Analyze Asteroid Threats" node:**  
   - Type: Code (JavaScript)  
   - Copy the JS logic that iterates asteroids, identifies hazards, biggest asteroid, and calculates alert level (LOW, MEDIUM, HIGH).  
   - Connect input from "Get NASA Asteroid Data".

6. **Create "Check for Hazardous Objects" node:**  
   - Type: If  
   - Condition: `hazardous_count` > 0  
   - Connect input from "Analyze Asteroid Threats".  
   - True output leads to next step (regional news).  
   - False output leads to "Create All-Clear Status".

7. **Create "Create All-Clear Status" node:**  
   - Type: Code  
   - Return JSON indicating no hazards detected with summary info.  
   - Connect input from false branch of "Check for Hazardous Objects".

8. **Create "Configure Regional Search" node:**  
   - Type: Code  
   - Generate three objects for US, Japan, EU with language, country, search suffix, and asteroid name.  
   - Connect input from true branch of "Check for Hazardous Objects".

9. **Create "Loop Over Regions" node:**  
   - Type: SplitInBatches  
   - Default batch size (1) to process each region query individually.  
   - Connect input from "Configure Regional Search".

10. **Create "Search Regional News" node:**  
    - Type: Apify  
    - Actor ID: Google Search Scraper (apify/google-search-scraper)  
    - Input JSON:  
      ```json
      {
        "queries": "{{ $('Loop Over Regions').item.json.search_name }}{{ $('Loop Over Regions').item.json.query_suffix }}",
        "maxPagesPerQuery": 1,
        "resultsPerPage": 5
      }
      ```  
    - Authentication: Apify OAuth2 credentials required.  
    - Connect input from "Loop Over Regions".  
    - Connect output back to "Loop Over Regions" to process next batch.

11. **Create "Aggregate All News" node:**  
    - Type: Aggregate  
    - Aggregate all item data into a single array under "news_results".  
    - Connect input from "Loop Over Regions" main output.

12. **Create "AI Fact-Check Analysis" node:**  
    - Type: OpenAI (LangChain)  
    - Model: gpt-4o-mini  
    - Input: aggregated news data.  
    - Requires OpenAI API credentials.

13. **Create "Parse AI Response" node:**  
    - Type: Code  
    - Parses AI JSON from text, extracts JSON inside code blocks, with fallback for parse errors.  
    - Connect input from "AI Fact-Check Analysis".

14. **Create "Generate Visualization Charts" node:**  
    - Type: Code  
    - Generates QuickChart.io URLs for threat gauge, panic gauge, and regional media tone bar chart.  
    - Connect input from "Parse AI Response".

15. **Create "Format Multi-Channel Messages" node:**  
    - Type: Code  
    - Formats Slack, Discord, Email messages including emojis, fact-check summaries, charts, misinformation alerts.  
    - Connect input from "Generate Visualization Charts".

16. **Create "Send Slack Alert" node:**  
    - Type: Slack  
    - Authentication: OAuth2 (Slack App credentials)  
    - Channel ID: specify target Slack channel  
    - Message: attachment type with color from alert level and chart image URL.  
    - Connect input from "Format Multi-Channel Messages".

17. **Create "Send Discord Alert" node:**  
    - Type: Discord  
    - Authentication: Webhook URL  
    - Content: formatted Discord message.  
    - Connect input from "Format Multi-Channel Messages".

18. **Create "Send Email Alert" node:**  
    - Type: Email Send  
    - Configure SMTP credentials and set sender and recipient emails.  
    - Subject and HTML body from formatted messages.  
    - Connect input from "Format Multi-Channel Messages".

19. **Create "Webhook Response" node:**  
    - Type: Respond to Webhook  
    - Respond with JSON indicating success, alert level, hazardous count, and threat score.  
    - Connect input from "Format Multi-Channel Messages".

20. **Create "Log to Google Sheets" node (optional):**  
    - Type: Google Sheets  
    - Operation: Append  
    - Configure Google Sheets credentials, document ID, sheet name.  
    - Map columns: Date, Alert Level, Threat Score, Top Asteroid, Hazardous Count, Media Panic Score, Most Accurate Region, Misinformation Detected.  
    - Connect input from "Format Multi-Channel Messages".

21. **Add Sticky Notes:**  
    - Add all sticky notes as per the workflow for instructions, setup, and explanations in appropriate positions.

22. **Connect all nodes as per described connections:**  
    - Merge Triggers → Get NASA Asteroid Data → Analyze Asteroid Threats → Check for Hazardous Objects  
    - Check for Hazardous Objects (true) → Configure Regional Search → Loop Over Regions → Search Regional News (recursive) → Aggregate All News → AI Fact-Check Analysis → Parse AI Response → Generate Visualization Charts → Format Multi-Channel Messages → [Send Slack, Discord, Email, Webhook Response, Log to Sheets]  
    - Check for Hazardous Objects (false) → Create All-Clear Status (ends workflow).

23. **Test workflow manually and via webhook.**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| NASA API key is required and free at https://api.nasa.gov/                                                             | Setup Instructions Sticky Note                   |
| Apify account needed for Google Search Scraper: https://apify.com/                                                     | Setup Instructions Sticky Note                   |
| OpenAI API key required from https://platform.openai.com/                                                              | Setup Instructions Sticky Note                   |
| Slack app OAuth2 credentials setup required with appropriate channel permissions                                        | Setup Instructions Sticky Note                    |
| Discord webhook URL needed for Discord alerts                                                                          | Setup Instructions Sticky Note                    |
| SMTP settings required for Email alerts                                                                                | Setup Instructions Sticky Note                    |
| Google Sheets optional for logging; requires a sheet with columns: Date, Alert Level, Hazardous Count, Threat Score, etc.| Setup Instructions Sticky Note                    |
| Workflow intended for space enthusiasts, journalists, researchers, educators interested in planetary defense          | Overall workflow description Sticky Note         |
| AI Fact-checking uses GPT-4o-mini model for balanced cost and capability                                               | AI Fact-Check Sticky Note                         |
| Media panic and threat scoring uses visual gauge charts from QuickChart.io                                              | Visualization Sticky Note                          |
| Multi-channel notifications can be customized by removing unused channel nodes                                         | Output Channels Sticky Note                        |

---

**Disclaimer:**  
This document and the workflow it describes are generated exclusively from an n8n automation. It complies with all content policies and processes only legal and public data. No illegal, offensive, or protected content is included.