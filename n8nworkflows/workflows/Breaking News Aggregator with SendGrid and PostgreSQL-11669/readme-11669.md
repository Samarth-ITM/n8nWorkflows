Breaking News Aggregator with SendGrid and PostgreSQL

https://n8nworkflows.xyz/workflows/breaking-news-aggregator-with-sendgrid-and-postgresql-11669


# Breaking News Aggregator with SendGrid and PostgreSQL

### 1. Workflow Overview

This workflow, titled **"Breaking News Aggregator with SendGrid and PostgreSQL"**, is designed to automate the daily aggregation, processing, and alerting of regulatory news updates from various government and financial regulatory sources. Its primary use case is to keep compliance teams continuously informed of new regulations that could impact their operations.

The workflow is structured into four main logical blocks:

- **1.1 Data Collection:** Triggered once every 24 hours, it defines and loops through a curated list of regulatory news sources, scraping updates from each.
- **1.2 Processing:** Normalizes the scraped data, scores the impact level based on keyword detection, and adds metadata like timestamps.
- **1.3 Alerts & Storage:** Routes high-impact updates to email alerts via SendGrid and stores all updates in a PostgreSQL database.
- **1.4 Workflow Overview & Documentation:** Provides embedded documentation and setup instructions within the workflow for maintainability and onboarding.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Collection

**Overview:**  
This block initiates the workflow daily, defines the regulatory news sources, and loops through each source to scrape relevant updates using AI-powered scraping.

**Nodes Involved:**  
- Daily Regulatory Check (Schedule Trigger)  
- Define Regulatory Sources (Code)  
- Loop Through Sources (SplitInBatches)  
- Scrape Regulatory Page (ScrapeGraphAI)  

**Node Details:**  

- **Daily Regulatory Check**  
  - *Type & Role:* Schedule Trigger, initiates workflow every 24 hours.  
  - *Configuration:* Interval set to 24 hours.  
  - *Inputs/Outputs:* No input; outputs trigger to "Define Regulatory Sources".  
  - *Failure Modes:* Misconfigured schedule, system time errors.  

- **Define Regulatory Sources**  
  - *Type & Role:* Code node that outputs a list of regulatory URLs to process.  
  - *Configuration:* Hardcoded array of 3 sources: SEC RSS feed, Federal Register, FCA news page.  
  - *Key Variables:* Outputs JSON objects with `url` properties.  
  - *Inputs:* Trigger from Schedule node.  
  - *Outputs:* Feeds URLs to "Loop Through Sources".  
  - *Failure Modes:* Syntax errors in code, empty source list.  

- **Loop Through Sources**  
  - *Type & Role:* SplitInBatches node, processes each source URL sequentially.  
  - *Configuration:* Default batch size (1) for sequential processing.  
  - *Inputs:* Receives list of URLs.  
  - *Outputs:* Sends one URL at a time to "Scrape Regulatory Page".  
  - *Failure Modes:* Batch size misconfiguration, node execution errors.  

- **Scrape Regulatory Page**  
  - *Type & Role:* ScrapeGraphAI node, scrapes regulatory updates from the provided URL.  
  - *Configuration:*  
    - `userPrompt`: Extracts updates with fields: title, summary, date, link, impactLevel (sets "Normal" if not inferred).  
    - `websiteUrl`: Dynamically set from current batch item URL (`={{ $json.url }}`).  
  - *Inputs:* Receives one URL from loop.  
  - *Outputs:* JSON objects representing extracted regulatory updates.  
  - *Failure Modes:* Connectivity issues, scraping failures, parsing errors, credential misconfiguration for ScrapeGraphAI.  
  - *Version:* v1  

---

#### 1.2 Processing

**Overview:**  
This block normalizes and scores the impact of each regulatory update, then appends timestamps and metadata before deciding how to route the data.

**Nodes Involved:**  
- Normalize & Score Impact (Code)  
- Add Timestamp & Metadata (Set)  
- High Impact? (IF)  

**Node Details:**  

- **Normalize & Score Impact**  
  - *Type & Role:* Code node that cleans and scores impact levels.  
  - *Configuration:*  
    - Uses regex to detect keywords indicating high impact (e.g., "effective immediately", "penalty", "urgent").  
    - Sets `impactLevel` to "High" if keywords present or if original impactLevel is 'high'.  
    - Fills missing fields with defaults (e.g., date to current ISO string).  
  - *Inputs:* Receives scraped items from "Scrape Regulatory Page".  
  - *Outputs:* JSON with normalized fields: title, summary, date, link, impactLevel, highImpact (boolean).  
  - *Failure Modes:* Regex errors, missing fields, empty input arrays.  

- **Add Timestamp & Metadata**  
  - *Type & Role:* Set node adding processing metadata.  
  - *Configuration:* Adds a `timestamp` field with current ISO datetime.  
  - *Inputs:* Normalized data from previous node.  
  - *Outputs:* Passes enriched data to "High Impact?" node.  
  - *Failure Modes:* Date generation errors (rare), data overwrite risks.  

- **High Impact?**  
  - *Type & Role:* IF node to branch logic based on `highImpact` boolean.  
  - *Configuration:* Condition checks if `highImpact` is true.  
  - *Inputs:* Receives data with `highImpact` flag.  
  - *Outputs:*  
    - True branch: proceeds to email preparation.  
    - False branch: skips email, goes directly to storage.  
  - *Failure Modes:* Expression evaluation errors, missing flag field.  

---

#### 1.3 Alerts & Storage

**Overview:**  
High-impact updates trigger email alerts through SendGrid, while all updates are stored in PostgreSQL for audit and reporting.

**Nodes Involved:**  
- Prepare Email Content (Set)  
- SendGrid Alert (SendGrid)  
- Store Regulation Record (PostgreSQL)  

**Node Details:**  

- **Prepare Email Content**  
  - *Type & Role:* Set node to prepare email body and metadata for alerts.  
  - *Configuration:* Sets email subject, recipient, and body content based on update data.  
  - *Inputs:* High-impact regulatory update from IF node.  
  - *Outputs:* Passes email data to SendGrid node.  
  - *Failure Modes:* Missing recipient or content fields, misformatted email content.  

- **SendGrid Alert**  
  - *Type & Role:* SendGrid node sends notification emails.  
  - *Configuration:* Uses configured SendGrid API credentials; sends to specified recipient(s).  
  - *Inputs:* Receives prepared email content.  
  - *Outputs:* Passes data downstream to storage node after sending.  
  - *Failure Modes:* Authentication failure, network errors, rate limiting, invalid email addresses.  

- **Store Regulation Record**  
  - *Type & Role:* PostgreSQL node inserts update records into `regulatory_updates` table.  
  - *Configuration:*  
    - Inserts fields: title, summary, high_impact, inserted_at, source_link, impact_level, effective_date.  
    - Uses defined mapping mode for explicit column mapping.  
    - Requires PostgreSQL credentials and pre-created table with appropriate schema.  
  - *Inputs:* Receives either from SendGrid (high-impact branch) or directly from IF node (normal-impact branch).  
  - *Outputs:* Final step, no further output.  
  - *Failure Modes:* Database connection errors, SQL errors, constraint violations, missing credentials.  
  - *Version:* v2.5  

---

#### 1.4 Workflow Overview & Documentation

**Overview:**  
This section contains sticky notes that document the workflow purpose, setup instructions, and segment grouping to assist maintainers and new users.

**Nodes Involved:**  
- Workflow Overview (Sticky Note)  
- Section - Data Collection (Sticky Note)  
- Section - Processing (Sticky Note)  
- Section - Alerts & Storage (Sticky Note)  

**Node Details:**  

- These nodes only contain markdown-formatted documentation content for clarity and guidance.  
- Positioned to visually group related nodes.  
- No inputs or outputs; purely informational.  
- Notes include setup steps, explanation of flow, and functional segmentation.

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                      |
|------------------------|---------------------------|--------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Daily Regulatory Check  | Schedule Trigger          | Trigger daily workflow          | -                           | Define Regulatory Sources     | ## Data Collection<br>Triggers daily and loops through regulatory URLs. ScrapeGraphAI extracts headlines, summaries, dates, and impact levels from each source. |
| Define Regulatory Sources | Code                     | Defines list of source URLs     | Daily Regulatory Check       | Loop Through Sources          | Same as above                                                                                                   |
| Loop Through Sources    | SplitInBatches            | Iterates through sources one by one | Define Regulatory Sources   | Scrape Regulatory Page        | Same as above                                                                                                   |
| Scrape Regulatory Page  | ScrapeGraphAI             | Scrapes regulatory updates      | Loop Through Sources         | Normalize & Score Impact      | Same as above                                                                                                   |
| Normalize & Score Impact| Code                      | Normalizes and scores impact    | Scrape Regulatory Page       | Add Timestamp & Metadata      | ## Processing<br>Normalizes data, scores impact using keyword detection, and adds timestamps before routing.   |
| Add Timestamp & Metadata| Set                       | Adds timestamp metadata         | Normalize & Score Impact     | High Impact?                  | Same as above                                                                                                   |
| High Impact?            | IF                        | Branches based on impact        | Add Timestamp & Metadata     | Prepare Email Content (true), Store Regulation Record (false) | ## Alerts & Storage<br>High-impact updates trigger SendGrid email alerts. All records are stored in PostgreSQL for reporting. |
| Prepare Email Content   | Set                       | Prepares email for alert        | High Impact? (true branch)   | SendGrid Alert               | Same as above                                                                                                   |
| SendGrid Alert         | SendGrid                   | Sends email alerts              | Prepare Email Content        | Store Regulation Record       | Same as above                                                                                                   |
| Store Regulation Record | PostgreSQL                 | Stores update in database       | High Impact? (false branch), SendGrid Alert | -                     | Same as above                                                                                                   |
| Workflow Overview       | Sticky Note                | Documentation & setup guide     | -                           | -                            | ## How it works... Setup steps included (see full note content).                                               |
| Section - Data Collection | Sticky Note              | Visual section label            | -                           | -                            | Same as "Data Collection" note                                                                                   |
| Section - Processing    | Sticky Note                | Visual section label            | -                           | -                            | Same as "Processing" note                                                                                        |
| Section - Alerts & Storage | Sticky Note             | Visual section label            | -                           | -                            | Same as "Alerts & Storage" note                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Node Type: Schedule Trigger  
   - Name: "Daily Regulatory Check"  
   - Parameters: Set interval to every 24 hours (hoursInterval = 24)  
   - No credentials required.  

2. **Create Code Node to Define Sources:**  
   - Node Type: Code  
   - Name: "Define Regulatory Sources"  
   - Code: Return array of JSON objects with URLs for regulatory sites, e.g.:  
     ```js
     return [
       {json: {url: 'https://www.sec.gov/news/pressreleases.rss'}},
       {json: {url: 'https://www.federalregister.gov/'}},
       {json: {url: 'https://www.fca.org.uk/news'}}
     ];
     ```  
   - Connect output of Schedule Trigger to this node.  

3. **Create SplitInBatches Node:**  
   - Node Type: SplitInBatches  
   - Name: "Loop Through Sources"  
   - Default batch size (1) to process sources sequentially.  
   - Connect output of "Define Regulatory Sources" to this node.  

4. **Create ScrapeGraphAI Node:**  
   - Node Type: ScrapeGraphAI  
   - Name: "Scrape Regulatory Page"  
   - Parameters:  
     - `userPrompt`: Extract regulatory updates with the specified JSON structure and impactLevel fallback.  
     - `websiteUrl`: Set expression to `={{ $json.url }}`  
   - Configure ScrapeGraphAI credentials in n8n settings.  
   - Connect output of "Loop Through Sources" to this node.  

5. **Create Code Node to Normalize and Score Impact:**  
   - Node Type: Code  
   - Name: "Normalize & Score Impact"  
   - Code logic:  
     - Use regex to detect urgent keywords in the summary.  
     - Set `impactLevel` to "High" if keywords found or original impactLevel is 'high', else "Normal".  
     - Fill missing fields with default values.  
   - Connect output of "Scrape Regulatory Page" to this node.  

6. **Create Set Node to Add Timestamp:**  
   - Node Type: Set  
   - Name: "Add Timestamp & Metadata"  
   - Parameters: Add a new field `timestamp` with current datetime via expression: `={{ new Date().toISOString() }}`  
   - Connect output of "Normalize & Score Impact" to this node.  

7. **Create IF Node to Branch on High Impact:**  
   - Node Type: IF  
   - Name: "High Impact?"  
   - Condition: Boolean check if `={{ $json.highImpact }}` is true.  
   - Connect output of "Add Timestamp & Metadata" to this node.  

8. **Create Set Node to Prepare Email Content:**  
   - Node Type: Set  
   - Name: "Prepare Email Content"  
   - Parameters: Define email fields such as subject, recipient (replace with your team’s email), and body content using data fields (title, summary, link, date).  
   - Connect the TRUE branch of "High Impact?" to this node.  

9. **Create SendGrid Node to Send Alerts:**  
   - Node Type: SendGrid  
   - Name: "SendGrid Alert"  
   - Configure SendGrid API credentials in n8n.  
   - Parameters: Use email fields from "Prepare Email Content" node.  
   - Connect output of "Prepare Email Content" to this node.  

10. **Create PostgreSQL Node to Store Records:**  
    - Node Type: PostgreSQL  
    - Name: "Store Regulation Record"  
    - Configure PostgreSQL credentials with access to a database containing a table `regulatory_updates` with columns:  
      - title (text), summary (text), high_impact (boolean), inserted_at (timestamp), source_link (text), impact_level (text), effective_date (date or text).  
    - Map columns explicitly with expressions pulling from current item JSON fields.  
    - Connect:  
      - FALSE branch of "High Impact?" directly to this node.  
      - Output of "SendGrid Alert" to this node (to store after sending alert).  

11. **Add Sticky Notes for Documentation:**  
    - Create sticky notes near relevant nodes with content describing each section and overall workflow setup steps:  
      - Workflow Overview (full description and setup instructions).  
      - Section labels: Data Collection, Processing, Alerts & Storage.  

12. **Activate Workflow:**  
    - Test each node step-by-step using manual trigger or partial executions.  
    - Verify credentials and connectivity for ScrapeGraphAI, SendGrid, and PostgreSQL.  
    - Activate workflow for daily automated execution.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow uses ScrapeGraphAI to extract structured regulatory news from heterogeneous web sources and RSS feeds. Ensure ScrapeGraphAI credentials are configured and have sufficient API quota.                                                                  | n8n ScrapeGraphAI node documentation                      |
| PostgreSQL database must have a table `regulatory_updates` with columns matching the node’s mapping. Sample schema creation is recommended before running.                                                                                                         | PostgreSQL setup best practices                            |
| SendGrid API key with email-sending permissions is required. Replace example recipient email with your compliance team’s address to receive alerts.                                                                                                                | SendGrid official API documentation                        |
| The keyword list and impact scoring logic in the "Normalize & Score Impact" node can be customized to reflect organizational priorities and regulatory language nuances.                                                                                            | Customization recommendation                               |
| Workflow documentation is embedded in sticky notes for transparency and ease of maintenance. Keep these updated when modifying the workflow.                                                                                                                      | Internal documentation practice                            |
| For improved reliability, consider adding error handling nodes or retry logic especially around ScrapeGraphAI and SendGrid nodes to handle transient network or API errors.                                                                                         | Reliability and error handling best practices             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. It strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.