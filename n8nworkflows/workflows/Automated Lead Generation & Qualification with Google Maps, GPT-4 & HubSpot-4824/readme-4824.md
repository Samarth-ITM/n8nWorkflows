Automated Lead Generation & Qualification with Google Maps, GPT-4 & HubSpot

https://n8nworkflows.xyz/workflows/automated-lead-generation---qualification-with-google-maps--gpt-4---hubspot-4824


# Automated Lead Generation & Qualification with Google Maps, GPT-4 & HubSpot

### 1. Workflow Overview

This workflow, titled **"Automated Lead Generation & Qualification with Google Maps, GPT-4 & HubSpot"**, automates the process of generating, cleansing, qualifying, enriching, and exporting sales leads. It integrates multiple data sources and services—Google Maps, Yellow Pages, AI-based lead qualification using GPT-4, Slack notifications, and HubSpot CRM—into a seamless pipeline. The workflow targets sales and marketing teams seeking to automate lead discovery and qualification to optimize outreach efforts and CRM updates.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception & Configuration**  
  Manual trigger and configuration setup to initiate the lead generation process.

- **1.2 Data Collection**  
  Scraping leads from Google Maps and Yellow Pages.

- **1.3 Data Cleaning & Verification**  
  Advanced cleaning of raw data and email verification.

- **1.4 AI-Based Lead Qualification**  
  Using GPT-4 (OpenAI) to assess lead quality and relevance.

- **1.5 Lead Enrichment & Analytics**  
  Further lead enrichment via custom logic and analytics generation.

- **1.6 Filtering & Export**  
  Filtering qualified leads, exporting them to Google Sheets, sending Slack alerts, and creating HubSpot contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration

- **Overview:**  
  This block starts the workflow manually and sets up the base configuration for subsequent nodes.

- **Nodes Involved:**  
  - ▶️ Start Lead Generation (Manual Trigger)  
  - 🔧 Configuration Hub (Set node)  
  - Pro Features Overview (Sticky Note)

- **Node Details:**

  1. **▶️ Start Lead Generation**  
     - *Type:* Manual Trigger  
     - *Role:* Entry point, manual activation of the lead generation process.  
     - *Configuration:* Default manual trigger, no parameters.  
     - *Input:* None  
     - *Output:* Connected to 🔧 Configuration Hub.  
     - *Edge Cases:* Workflow not starting if user forgets to trigger manually.

  2. **🔧 Configuration Hub**  
     - *Type:* Set  
     - *Role:* Holds configuration variables and parameters used by scrapers and other nodes.  
     - *Configuration:* Empty parameters in JSON, but typically used to define search criteria, API keys, or flags.  
     - *Input:* From ▶️ Start Lead Generation  
     - *Output:* Feeds the two scraper nodes and lead enrichment node.  
     - *Edge Cases:* Misconfiguration here can lead to errors or empty results in scrapers.

  3. **Pro Features Overview**  
     - *Type:* Sticky Note  
     - *Role:* Provides user instructions or product feature highlights (currently empty).  
     - *Input/Output:* Visual aid only.

#### 2.2 Data Collection

- **Overview:**  
  Scrapes lead data from Google Maps and Yellow Pages using HTTP requests, then forwards raw data for cleaning.

- **Nodes Involved:**  
  - 🗺️ Google Maps Scraper (HTTP Request)  
  - 📞 Yellow Pages Scraper (HTTP Request)  

- **Node Details:**

  1. **🗺️ Google Maps Scraper**  
     - *Type:* HTTP Request  
     - *Role:* Queries Google Maps API or web endpoints to collect business lead information.  
     - *Configuration:* Likely configured with URL, query parameters (location, keywords), and headers; authentication may be required.  
     - *Input:* From 🔧 Configuration Hub  
     - *Output:* To 🧹 Advanced Data Cleaner  
     - *Edge Cases:* API rate limits, network errors, or invalid query parameters.

  2. **📞 Yellow Pages Scraper**  
     - *Type:* HTTP Request  
     - *Role:* Similar to Google Maps Scraper but targets Yellow Pages listings.  
     - *Configuration:* URL, search parameters, and possibly proxy or user-agent headers for scraping.  
     - *Input:* From 🔧 Configuration Hub  
     - *Output:* To 🧹 Advanced Data Cleaner  
     - *Edge Cases:* IP blocking, scraping protection, data format changes.

#### 2.3 Data Cleaning & Verification

- **Overview:**  
  Cleans and standardizes raw scraped data, then verifies email addresses to ensure valid contacts.

- **Nodes Involved:**  
  - 🧹 Advanced Data Cleaner (Code)  
  - ✉️ Email Verification (HTTP Request)

- **Node Details:**

  1. **🧹 Advanced Data Cleaner**  
     - *Type:* Code (JavaScript)  
     - *Role:* Custom script to clean, normalize, and filter raw lead data (e.g., removing duplicates, formatting phone numbers).  
     - *Configuration:* Script logic embedded in node; can handle array manipulation and field validation.  
     - *Input:* From both scrapers  
     - *Output:* To ✉️ Email Verification  
     - *Edge Cases:* Unexpected data formats causing script errors; empty or malformed input data.

  2. **✉️ Email Verification**  
     - *Type:* HTTP Request  
     - *Role:* Calls an external email verification API to validate email addresses for deliverability and existence.  
     - *Configuration:* URL, API key in headers or query parameters, email address passed dynamically.  
     - *Input:* From 🧹 Advanced Data Cleaner  
     - *Output:* To 🤖 AI Lead Qualification  
     - *Edge Cases:* API quota limits, invalid/missing emails, network issues.

#### 2.4 AI-Based Lead Qualification

- **Overview:**  
  Uses GPT-4 via LangChain OpenAI integration to qualitatively assess leads based on enriched data and email verification results.

- **Nodes Involved:**  
  - 🤖 AI Lead Qualification (OpenAI)

- **Node Details:**

  1. **🤖 AI Lead Qualification**  
     - *Type:* OpenAI Language Model (LangChain node)  
     - *Role:* Processes lead data through GPT-4 to score or classify lead quality.  
     - *Configuration:* OpenAI credentials required; prompt templates likely include lead details and qualification criteria.  
     - *Input:* From ✉️ Email Verification  
     - *Output:* To 💎 Lead Enrichment Engine  
     - *Edge Cases:* API key limits, model timeout, prompt errors, rate limiting.

#### 2.5 Lead Enrichment & Analytics

- **Overview:**  
  Enriches qualified leads with additional data and generates analytics for reporting.

- **Nodes Involved:**  
  - 💎 Lead Enrichment Engine (Code)  
  - 📈 Analytics Engine (Code)  
  - 📊 Export Analytics (Google Sheets)

- **Node Details:**

  1. **💎 Lead Enrichment Engine**  
     - *Type:* Code (JavaScript)  
     - *Role:* Adds or computes additional lead information, such as social profiles, company size, or lead scoring metrics.  
     - *Configuration:* Custom script logic embedded; may call external APIs or parse data fields.  
     - *Input:* From 🤖 AI Lead Qualification and 🔧 Configuration Hub  
     - *Output:* Three branches:  
       - 🎯 Quality Filter (for filtering leads)  
       - 📋 Export All Leads (exports raw enriched leads)  
       - 📈 Analytics Engine (for analytics calculations)  
     - *Edge Cases:* Script errors, missing enrichment data, API failures.

  2. **📈 Analytics Engine**  
     - *Type:* Code (JavaScript)  
     - *Role:* Aggregates metrics and KPIs from lead data for reporting purposes (e.g., total leads, conversion rates).  
     - *Configuration:* Embedded code to calculate statistics.  
     - *Input:* From 💎 Lead Enrichment Engine  
     - *Output:* To 📊 Export Analytics  
     - *Edge Cases:* Empty data sets, calculation errors.

  3. **📊 Export Analytics**  
     - *Type:* Google Sheets  
     - *Role:* Writes analytics data to a Google Sheet for monitoring and reporting.  
     - *Configuration:* Google Sheets credential required, target spreadsheet and sheet specified.  
     - *Input:* From 📈 Analytics Engine  
     - *Output:* None  
     - *Edge Cases:* Authentication issues, sheet access permissions.

#### 2.6 Filtering & Export

- **Overview:**  
  Filters leads based on quality, exports qualified leads, sends Slack notifications, and creates HubSpot contacts.

- **Nodes Involved:**  
  - 🎯 Quality Filter (If)  
  - 📊 Export Qualified Leads (Google Sheets)  
  - 🔔 Slack Alert (Slack)  
  - 🏢 Create HubSpot Contact (HubSpot)  
  - 📋 Export All Leads (Google Sheets)

- **Node Details:**

  1. **🎯 Quality Filter**  
     - *Type:* If  
     - *Role:* Checks lead data against qualification criteria to separate qualified and unqualified leads.  
     - *Configuration:* Conditional expression based on lead scores or flags.  
     - *Input:* From 💎 Lead Enrichment Engine  
     - *Output:* Qualified leads routed to Export Qualified Leads and Slack Alert  
     - *Edge Cases:* Incorrect filter logic leading to false positives/negatives.

  2. **📊 Export Qualified Leads**  
     - *Type:* Google Sheets  
     - *Role:* Exports only qualified leads to a dedicated Google Sheet tab.  
     - *Configuration:* Google Sheets credentials, spreadsheet and sheet name configured.  
     - *Input:* From 🎯 Quality Filter (true branch)  
     - *Output:* To 🏢 Create HubSpot Contact  
     - *Edge Cases:* Permission issues, quota limits.

  3. **🔔 Slack Alert**  
     - *Type:* Slack  
     - *Role:* Sends notification alerts to a Slack channel about newly qualified leads.  
     - *Configuration:* Slack OAuth2 credentials, webhook or bot token set, message template defined.  
     - *Input:* From 🎯 Quality Filter (true branch)  
     - *Output:* None  
     - *Edge Cases:* Slack API rate limits, authentication errors.

  4. **🏢 Create HubSpot Contact**  
     - *Type:* HubSpot  
     - *Role:* Creates or updates contacts in HubSpot CRM with qualified lead data.  
     - *Configuration:* HubSpot OAuth2 credentials, contact properties mapped.  
     - *Input:* From 📊 Export Qualified Leads  
     - *Output:* None  
     - *Edge Cases:* API errors, duplicate contacts, permission restrictions.

  5. **📋 Export All Leads**  
     - *Type:* Google Sheets  
     - *Role:* Exports all enriched leads regardless of qualification status for record-keeping.  
     - *Configuration:* Google Sheets credentials, spreadsheet and sheet name configured.  
     - *Input:* From 💎 Lead Enrichment Engine (second output)  
     - *Output:* None  
     - *Edge Cases:* Same as other Google Sheets nodes.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                     | Input Node(s)                         | Output Node(s)                            | Sticky Note                                                          |
|--------------------------|---------------------------|-----------------------------------|-------------------------------------|------------------------------------------|----------------------------------------------------------------------|
| Pro Features Overview     | Sticky Note               | User instructions / overview      | None                                | None                                     |                                                                      |
| ▶️ Start Lead Generation   | Manual Trigger            | Workflow entry point              | None                                | 🔧 Configuration Hub                     |                                                                      |
| 🔧 Configuration Hub       | Set                       | Configuration & parameters setup | ▶️ Start Lead Generation             | 🗺️ Google Maps Scraper, 📞 Yellow Pages Scraper, 💎 Lead Enrichment Engine |                                                                      |
| 🗺️ Google Maps Scraper     | HTTP Request              | Scrape Google Maps leads          | 🔧 Configuration Hub                 | 🧹 Advanced Data Cleaner                 |                                                                      |
| 📞 Yellow Pages Scraper    | HTTP Request              | Scrape Yellow Pages leads         | 🔧 Configuration Hub                 | 🧹 Advanced Data Cleaner                 |                                                                      |
| 🧹 Advanced Data Cleaner   | Code                      | Clean & normalize raw data        | 🗺️ Google Maps Scraper, 📞 Yellow Pages Scraper | ✉️ Email Verification                   |                                                                      |
| ✉️ Email Verification      | HTTP Request              | Verify email addresses            | 🧹 Advanced Data Cleaner             | 🤖 AI Lead Qualification                 |                                                                      |
| 🤖 AI Lead Qualification   | OpenAI (LangChain)        | AI-based lead quality scoring     | ✉️ Email Verification                | 💎 Lead Enrichment Engine                 |                                                                      |
| 💎 Lead Enrichment Engine  | Code                      | Enrich leads and trigger branches | 🤖 AI Lead Qualification, 🔧 Configuration Hub | 🎯 Quality Filter, 📋 Export All Leads, 📈 Analytics Engine |                                                                      |
| 🎯 Quality Filter          | If                        | Filter qualified leads            | 💎 Lead Enrichment Engine            | 📊 Export Qualified Leads, 🔔 Slack Alert |                                                                      |
| 📊 Export Qualified Leads  | Google Sheets             | Export qualified leads            | 🎯 Quality Filter                   | 🏢 Create HubSpot Contact                 |                                                                      |
| 🔔 Slack Alert             | Slack                     | Notify Slack channel of leads    | 🎯 Quality Filter                   | None                                     |                                                                      |
| 🏢 Create HubSpot Contact  | HubSpot                   | Create/update HubSpot contacts    | 📊 Export Qualified Leads            | None                                     |                                                                      |
| 📋 Export All Leads        | Google Sheets             | Export all enriched leads         | 💎 Lead Enrichment Engine            | None                                     |                                                                      |
| 📈 Analytics Engine        | Code                      | Generate analytics data           | 💎 Lead Enrichment Engine            | 📊 Export Analytics                      |                                                                      |
| 📊 Export Analytics        | Google Sheets             | Export analytics data             | 📈 Analytics Engine                  | None                                     |                                                                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a manual trigger node named **"▶️ Start Lead Generation"**. This will be the workflow entry point.

**Step 2:** Create a **Set** node named **"🔧 Configuration Hub"**. Configure this node with all required parameters such as search keywords, locations, API keys for scraping, and any flags for controlling flow. Connect **Start Lead Generation** → **Configuration Hub**.

**Step 3:** Add two **HTTP Request** nodes:

- **"🗺️ Google Maps Scraper"**: Configure to query Google Maps API or scrape relevant data. Use query parameters and authentication as needed. Connect **Configuration Hub** → **Google Maps Scraper**.

- **"📞 Yellow Pages Scraper"**: Configure similarly to scrape Yellow Pages data, including headers or proxies if scraping protected sites. Connect **Configuration Hub** → **Yellow Pages Scraper**.

**Step 4:** Create a **Code** node named **"🧹 Advanced Data Cleaner"**. Write JavaScript to process incoming data arrays from both scrapers, normalizing fields, deduplicating, and filtering invalid entries. Connect both scraper nodes to this node.

**Step 5:** Add an **HTTP Request** node named **"✉️ Email Verification"** configured to call an email validation API. Pass email addresses dynamically from the cleaned data. Connect **Advanced Data Cleaner** → **Email Verification**.

**Step 6:** Add an **OpenAI (LangChain)** node named **"🤖 AI Lead Qualification"**. Configure with OpenAI credentials and define prompt templates that use lead details to score or classify leads. Connect **Email Verification** → **AI Lead Qualification**.

**Step 7:** Create another **Code** node named **"💎 Lead Enrichment Engine"**. Implement scripts to enrich leads with additional data (e.g., social media, company size), and prepare outputs for filtering, exporting all leads, and analytics. Connect **AI Lead Qualification** and **Configuration Hub** → **Lead Enrichment Engine**.

**Step 8:** Add an **If** node named **"🎯 Quality Filter"**. Define conditions based on enrichment and AI qualification results to separate qualified leads. Connect **Lead Enrichment Engine** → **Quality Filter**.

**Step 9:** Add a **Google Sheets** node named **"📊 Export Qualified Leads"**. Configure with Google credentials and specify the spreadsheet/sheet to store qualified leads. Connect **Quality Filter (true branch)** → **Export Qualified Leads**.

**Step 10:** Add a **Slack** node named **"🔔 Slack Alert"**. Configure Slack OAuth2 credentials and define the message template to alert about new qualified leads. Connect **Quality Filter (true branch)** → **Slack Alert**.

**Step 11:** Add a **HubSpot** node named **"🏢 Create HubSpot Contact"**. Configure with HubSpot OAuth2 credentials to create or update contacts using exported lead data. Connect **Export Qualified Leads** → **Create HubSpot Contact**.

**Step 12:** Add a **Google Sheets** node named **"📋 Export All Leads"**. Configure to export all enriched leads regardless of qualification. Connect **Lead Enrichment Engine (output 2)** → **Export All Leads**.

**Step 13:** Add a **Code** node named **"📈 Analytics Engine"**. Write code to calculate KPIs and analytics from enriched leads. Connect **Lead Enrichment Engine (output 3)** → **Analytics Engine**.

**Step 14:** Add a **Google Sheets** node named **"📊 Export Analytics"**. Configure to export analytics results to a dedicated sheet. Connect **Analytics Engine** → **Export Analytics**.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                           |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow integrates Google Maps scraping with AI-based lead qualification using GPT-4            | Workflow title and description                             |
| Slack notifications allow real-time alerts on lead qualification status                          | Slack node usage                                           |
| HubSpot integration requires OAuth2 credentials and proper field mapping                         | HubSpot node setup                                         |
| Google Sheets nodes require enabling Google Drive API and setting correct spreadsheet permissions | Google Sheets node setup                                   |
| Email verification depends on external API services; ensure quota and credentials                | Email Verification node                                    |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies with all content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.