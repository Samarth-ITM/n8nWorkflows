Track Real Estate Market Pain Points with Apify, GPT-4o and Telegram Alerts

https://n8nworkflows.xyz/workflows/track-real-estate-market-pain-points-with-apify--gpt-4o-and-telegram-alerts-5984


# Track Real Estate Market Pain Points with Apify, GPT-4o and Telegram Alerts

### 1. Workflow Overview

This workflow, titled **"AI-Powered Real Estate Market Radar & Pain Point Detector,"** automates the monitoring of real estate market signals by scraping Google search results, extracting key pain points using AI, comparing them with historical data, and sending alerts via Telegram. It also stores the insights in Airtable for longitudinal tracking.

**Target Use Cases:**  
- Real estate agencies and PropTech companies seeking competitive intelligence  
- Sales and marketing teams requiring automated market trend detection  
- Freelancers and AI consultants building market intelligence tools  
- Any stakeholder needing daily automated tracking of real estate market pain points  

**Logical Blocks:**

- **1.1 Scheduling & Data Collection:** Triggers the workflow on a schedule and scrapes Google search results with Apify.  
- **1.2 Data Processing & AI Analysis:** Summarizes scraped results and extracts top 3 pain points using GPT-4o AI.  
- **1.3 Historical Data Retrieval:** Reads yesterday’s pain points from Airtable for comparison.  
- **1.4 Comparison & Trend Detection:** Compares today’s extracted pain points with yesterday’s to identify new and recurring issues.  
- **1.5 Notification & Storage:** Sends Telegram alerts with detected pain points and stores today’s data in Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Data Collection

- **Overview:**  
  This block initiates the workflow at a defined schedule and collects relevant real estate market data by scraping Google search results via Apify.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Apify Scraper

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow periodically (default daily interval)  
    - Configuration: Uses default interval (daily) trigger; can be customized for frequency  
    - Inputs: None  
    - Outputs: Triggers the Apify Scraper node  
    - Edge cases: Misconfiguration of scheduling can cause missed or overlapping runs

  - **Apify Scraper**  
    - Type: HTTP Request  
    - Role: Calls the Apify Google Search Scraper API synchronously to retrieve search result data  
    - Configuration:  
      - URL includes an Apify API token placeholder (`YOUR_APIFY_TOKEN`) that must be replaced with a valid token  
      - Runs `apify~google-search-scraper` actor synchronously and fetches dataset items  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Passes raw search results to the next node (Extract Summary)  
    - Edge cases: API token invalidity, network errors, rate limiting by Apify, empty or malformed response  

#### 1.2 Data Processing & AI Analysis

- **Overview:**  
  This block processes the raw search results by summarizing titles and descriptions, then applies an AI model (GPT-4o) to extract the top 3 real estate pain points.

- **Nodes Involved:**  
  - Extract Summary  
  - OpenAI Chat Model  
  - Extract Pain Points

- **Node Details:**

  - **Extract Summary**  
    - Type: Code (JavaScript)  
    - Role: Converts raw search results into a summarized string list of titles and descriptions  
    - Configuration:  
      - Maps over `organicResults` from Apify output  
      - Creates a numbered list in the format: `1. Title — Description` joined by double newlines  
    - Inputs: Apify Scraper JSON output  
    - Outputs: JSON with a `summary` field containing the compiled string  
    - Edge cases: Missing or empty `organicResults` array, malformed input JSON causing code errors  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the GPT-4o AI model environment for extraction  
    - Configuration:  
      - Model set to `gpt-4o-mini`  
      - Credentials: Requires a valid OpenAI API key (`openAiApi` credential)  
    - Inputs: None directly; connected as the AI language model for "Extract Pain Points" node  
    - Outputs: Used internally by the LangChain agent node  
    - Edge cases: API quota limits, invalid credentials, network issues  

  - **Extract Pain Points**  
    - Type: LangChain Agent  
    - Role: Uses GPT-4o to analyze the summary text and extract the top 3 real estate market pain points  
    - Configuration:  
      - Input text expression: `={{ $json.summary }}`  
      - System message instructs AI to act as a market researcher extracting pain points explicitly or implicitly mentioned  
      - Uses the defined OpenAI Chat Model  
    - Inputs: Summary text from Extract Summary node  
    - Outputs: Extracted pain points text output  
    - Edge cases: AI model misinterpretation, empty summary input, API failures  

#### 1.3 Historical Data Retrieval

- **Overview:**  
  Retrieves yesterday’s recorded pain points from Airtable to enable comparison with today’s extracted data.

- **Nodes Involved:**  
  - Read Yesterday Pain Points

- **Node Details:**

  - **Read Yesterday Pain Points**  
    - Type: Airtable node (Read operation)  
    - Role: Fetches the last stored summary of pain points from Airtable  
    - Configuration:  
      - Table ID placeholder `YOUR_TABLE_ID` must be replaced with actual Airtable table reference  
      - Uses Airtable API token credential (`airtableTokenApi`)  
      - Assumes data includes a field named "Summary" containing previous pain points as newline-separated text  
    - Inputs: None (runs independently)  
    - Outputs: JSON with yesterday’s summary for comparison  
    - Edge cases: Incorrect table ID, expired or invalid Airtable token, empty or missing data fields, rate limits  

#### 1.4 Comparison & Trend Detection

- **Overview:**  
  Compares today’s pain points with yesterday’s to identify which are new and which are recurring, categorizing market trends.

- **Nodes Involved:**  
  - Compare Pain Points

- **Node Details:**

  - **Compare Pain Points**  
    - Type: Code (JavaScript)  
    - Role: Splits pain points text into arrays and filters for new vs recurring items  
    - Configuration:  
      - Splits today’s and yesterday’s pain points by newline and trims whitespace  
      - Creates two arrays: `newPainPoints` (unique today) and `recurring` (present in both days)  
    - Inputs:  
      - Today’s data from Extract Pain Points (`$json.output`)  
      - Yesterday’s data from Read Yesterday Pain Points (`$items('Read Yesterday Pain Points')[0].json.Summary`)  
    - Outputs: JSON object with arrays for `newPainPoints` and `recurring`  
    - Edge cases: Empty inputs, formatting inconsistencies, missing yesterday data leading to false positives  

#### 1.5 Notification & Storage

- **Overview:**  
  Sends a Telegram alert to notify stakeholders of new and recurring pain points, and stores today’s extracted data in Airtable for future comparison.

- **Nodes Involved:**  
  - Telegram Notifier  
  - Store to Airtable

- **Node Details:**

  - **Telegram Notifier**  
    - Type: Telegram node  
    - Role: Sends a formatted message to a specified Telegram chat with the pain points summary  
    - Configuration:  
      - Uses chat ID placeholder `YOUR_TELEGRAM_CHAT_ID`  
      - Message includes new and recurring pain points, formatted with bullet points  
    - Inputs: Output from Compare Pain Points node  
    - Outputs: None (side-effect node)  
    - Edge cases: Invalid chat ID, Telegram API errors, network failures  

  - **Store to Airtable**  
    - Type: Airtable node (Create operation)  
    - Role: Saves the extracted pain points summary for the current day to Airtable  
    - Configuration:  
      - Table ID placeholder `YOUR_TABLE_ID`  
      - Uses same Airtable API credentials as Read node  
      - Stores new pain points summary as a record for future retrieval  
    - Inputs: Output from Extract Pain Points node  
    - Outputs: Confirmation of record creation  
    - Edge cases: Airtable API errors, data schema mismatches, credential issues  

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                    | Input Node(s)          | Output Node(s)              | Sticky Note                                                                                         |
|-------------------------|------------------------------------|----------------------------------|-----------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                   | Initiates workflow on schedule   | None                  | Apify Scraper              |                                                                                                   |
| Apify Scraper           | HTTP Request                      | Scrapes Google search data       | Schedule Trigger      | Extract Summary            |                                                                                                   |
| Extract Summary         | Code                             | Summarizes search results        | Apify Scraper          | Extract Pain Points        |                                                                                                   |
| OpenAI Chat Model       | LangChain OpenAI Chat Model       | Provides GPT-4o AI environment   | None (AI model node)   | Used by Extract Pain Points|                                                                                                   |
| Extract Pain Points     | LangChain Agent                   | Extracts top 3 pain points via AI| Extract Summary        | Store to Airtable, Compare Pain Points |                                                                                                   |
| Read Yesterday Pain Points | Airtable (Read)                   | Fetches yesterday’s pain points  | None                  | Compare Pain Points         |                                                                                                   |
| Compare Pain Points     | Code                             | Compares today vs yesterday points| Extract Pain Points, Read Yesterday Pain Points | Telegram Notifier          |                                                                                                   |
| Telegram Notifier       | Telegram                         | Sends alert message on Telegram  | Compare Pain Points    | None                      |                                                                                                   |
| Store to Airtable       | Airtable (Create)                 | Stores today’s pain points       | Extract Pain Points    | None                      |                                                                                                   |
| Sticky Note             | Sticky Note                      | Documentation and comments       | None                  | None                      | ## Flow                                                                                           |
| Sticky Note1            | Sticky Note                      | Documentation reference          | None                  | None                      | ## Engine                                                                                         |
| Sticky Note2            | Sticky Note                      | Documentation reference          | None                  | None                      | ## Sinyal                                                                                         |
| Sticky Note3            | Sticky Note                      | Documentation reference          | None                  | None                      | ## Database                                                                                       |
| Sticky Note4            | Sticky Note                      | Documentation reference          | None                  | None                      | ## Get yesterday data                                                                             |
| Sticky Note5            | Sticky Note                      | Workflow detailed description    | None                  | None                      | # 🧠 AI-Powered Real Estate Market Radar & Pain Point Detector... (full intro and scope text)     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set the trigger interval to daily (default) or desired frequency.

2. **Create an HTTP Request node named "Apify Scraper"**  
   - Method: GET  
   - URL: `https://api.apify.com/v2/acts/apify~google-search-scraper/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`  
   - Replace `YOUR_APIFY_TOKEN` with a valid Apify API token.  
   - Connect Schedule Trigger → Apify Scraper.

3. **Create a Code node named "Extract Summary"**  
   - Paste this JavaScript code to map and summarize the search results:
     ```js
     return [{
       json: {
         summary: $json.organicResults.map((item, i) => `${i + 1}. ${item.title} — ${item.description}`).join('\n\n')
       }
     }];
     ```
   - Connect Apify Scraper → Extract Summary.

4. **Create a LangChain OpenAI Chat Model node named "OpenAI Chat Model"**  
   - Select model: `gpt-4o-mini`  
   - Configure credentials with a valid OpenAI API key.  
   - No direct input connections.

5. **Create a LangChain Agent node named "Extract Pain Points"**  
   - Set parameter `text` to expression: `={{ $json.summary }}`  
   - Under options, insert a System Message:  
     `"You are an AI market researcher. Analyze the text and extract the top 3 pain points real estate agents face. Only include what’s directly or implicitly mentioned."`  
   - Link the AI language model to the "OpenAI Chat Model" node.  
   - Connect Extract Summary → Extract Pain Points.

6. **Create an Airtable Read node named "Read Yesterday Pain Points"**  
   - Select your Airtable base and table (replace `YOUR_TABLE_ID`).  
   - Use an Airtable API token credential with read permissions.  
   - No input connections.

7. **Create a Code node named "Compare Pain Points"**  
   - Paste this JavaScript to compare today’s and yesterday’s pain points:
     ```js
     const today = $json.output.split('\n').map(p => p.trim());
     const yesterday = $items('Read Yesterday Pain Points')[0].json.Summary.split('\n').map(p => p.trim());

     const newPoints = today.filter(p => !yesterday.includes(p));
     const recurring = today.filter(p => yesterday.includes(p));

     return [{ json: { newPainPoints: newPoints, recurring } }];
     ```
   - Connect Extract Pain Points → Compare Pain Points (main input)  
   - Connect Read Yesterday Pain Points → Compare Pain Points (second input).

8. **Create a Telegram node named "Telegram Notifier"**  
   - Configure Telegram credentials or Bot Token.  
   - Set `chatId` to your Telegram chat ID (`YOUR_TELEGRAM_CHAT_ID`).  
   - Message text:
     ```
     📊 Real Estate Radar
     New Pain Points:
     ={{ $json.newPainPoints.join('\n') }}
     Recurring:
     ={{ $json.recurring.join('\n') }}
     ```
   - Connect Compare Pain Points → Telegram Notifier.

9. **Create an Airtable Create node named "Store to Airtable"**  
   - Configure to write to the same Airtable base/table as the Read node.  
   - Use the same Airtable API token credentials.  
   - Map fields to store the current pain points summary output from Extract Pain Points.  
   - Connect Extract Pain Points → Store to Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| # 🧠 AI-Powered Real Estate Market Radar & Pain Point Detector  \n\n## ❗️Problem  \nReal estate professionals often miss market signals due to lack of consistent monitoring and analysis. Manual tracking is time-consuming and prone to omission, risking lost leads and revenue.  \n\n## ✅ Solution  \nAutomates daily AI-powered market radar to scrape Google, extract pain points with GPT-4o, compare trends, alert via Telegram, and log to Airtable.  \n\n## 🧭 Scope  \nIncludes scraping, AI analysis, trend detection, notifications, and database logging, ready for extensions like cold emails and CRM sync.  \n\n## 👥 For Who  \nReal estate agencies, PropTech founders, sales/marketing teams, freelancers, and AI consultants needing market intelligence automation. | Sticky Note5 (attached to workflow overview) |
| ## Flow                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note (near Schedule Trigger)  |
| ## Engine                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note1 (near AI nodes)          |
| ## Sinyal                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note2 (near notification node)|
| ## Database                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note3 (near Airtable nodes)    |
| ## Get yesterday data                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note4 (near Read Yesterday Pain Points) |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects prevailing content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.