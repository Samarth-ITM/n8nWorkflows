Monitor Amazon Market Intelligence with ScrapeGraphAI to Google Docs

https://n8nworkflows.xyz/workflows/monitor-amazon-market-intelligence-with-scrapegraphai-to-google-docs-6974


# Monitor Amazon Market Intelligence with ScrapeGraphAI to Google Docs

---

### 1. Workflow Overview

This n8n workflow automates Amazon market intelligence gathering and reporting, designed for daily execution to provide data-driven competitive insights and strategic recommendations. It integrates web data scraping, data analysis, SEO keyword extraction, pricing strategy formulation, and automated reporting into Google Docs.

**Target Use Cases:**  
- E-commerce product managers seeking daily updates on Amazon market trends  
- Competitive intelligence teams analyzing pricing and product performance  
- Marketing teams identifying keyword opportunities for SEO and PPC  
- Strategic planners requiring automated, actionable reports

**Logical Blocks:**

- **1.1 Input Reception**  
  Scheduled trigger to start the workflow daily at 6:00 AM UTC.

- **1.2 Data Acquisition**  
  Scraping Amazon product listings via ScrapeGraphAI for specified category data.

- **1.3 Data Analysis**  
  Processing scraped data for pricing, quality, product rankings, and market gaps.

- **1.4 SEO & Keyword Strategy**  
  Extracting and analyzing keywords from top products to identify SEO opportunities.

- **1.5 Pricing Strategy Formulation**  
  Creating pricing recommendations based on market data and keyword insights.

- **1.6 Reporting**  
  Generating a comprehensive strategic report consolidating all analyses.

- **1.7 Google Docs Integration**  
  Exporting the report into a formatted Google Document for distribution and archival.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Triggers the workflow execution automatically once daily at 6:00 AM UTC.

- **Nodes Involved:**  
  - Daily Schedule

- **Node Details:**

  - **Daily Schedule**  
    - **Type & Role:** Schedule Trigger node, initiates the workflow on a fixed cron schedule.  
    - **Configuration:** Cron expression set to `0 6 * * *` (6:00 AM UTC daily).  
    - **Input/Output:** No input; output triggers subsequent scraping node.  
    - **Failure Modes:** Cron misconfiguration or workflow disabled could prevent execution.  
    - **Version:** 1.2  

---

#### 1.2 Data Acquisition

- **Overview:**  
  Scrapes Amazon product data using ScrapeGraphAI API to collect key product attributes.

- **Nodes Involved:**  
  - Amazon Product Scraper

- **Node Details:**

  - **Amazon Product Scraper**  
    - **Type & Role:** ScrapeGraphAI node; extracts product info from Amazon search results.  
    - **Configuration:**
      - Target URL set to Amazon search for "wireless bluetooth headphones".  
      - User prompt specifies data extraction fields: `title, price, rating, review_count, asin, prime_eligible, image_url`.  
      - Returns JSON array of products.  
    - **Input/Output:** Triggered by schedule node; outputs JSON product list to analyzer.  
    - **Failures/Edge Cases:**  
      - API key misconfiguration  
      - Amazon URL or page layout changes breaking scraping  
      - Rate limiting or IP blocking by Amazon  
      - Network timeouts  
    - **Version:** 1  

---

#### 1.3 Data Analysis

- **Overview:**  
  Analyzes scraped product data for pricing distribution, rating quality, top performers, and market opportunities.

- **Nodes Involved:**  
  - Product Analyzer

- **Node Details:**

  - **Product Analyzer (Code node)**  
    - **Type & Role:** JavaScript Code node performing statistical and qualitative analysis.  
    - **Configuration:**  
      - Parses input product array.  
      - Computes average, min, max prices; price segmentation (budget < $50, mid $50–150, premium > $150).  
      - Calculates rating averages and distribution by quality tiers.  
      - Identifies top 5 products by combined rating and review volume.  
      - Detects market gaps (quality issues, premium skew).  
    - **Key Expressions:**  
      - Parses numeric fields safely removing currency symbols and commas.  
      - Uses logs for ranking top products.  
    - **Input/Output:** Receives product JSON; outputs analysis JSON to Keyword Analyzer.  
    - **Failures/Edge Cases:**  
      - Empty or malformed input array  
      - Missing price or rating fields  
      - Division by zero if no valid prices or ratings  
    - **Version:** 2  

---

#### 1.4 SEO & Keyword Strategy

- **Overview:**  
  Extracts and analyzes keywords from top products to identify SEO opportunities and competitive keyword gaps.

- **Nodes Involved:**  
  - Keyword Analyzer

- **Node Details:**

  - **Keyword Analyzer (Code node)**  
    - **Type & Role:** JavaScript Code node for keyword frequency analysis and SEO insight generation.  
    - **Configuration:**  
      - Extracts words from top product titles.  
      - Filters stop words (like "with", "for", "wireless").  
      - Counts keyword frequency, classifies potential (High/Medium).  
      - Maps market opportunities to keyword suggestions.  
      - Detects market trends (premium/value focus).  
      - Provides actionable SEO recommendations.  
    - **Input/Output:** Input from Product Analyzer; outputs keyword analysis JSON to Pricing Strategy node.  
    - **Failures/Edge Cases:**  
      - No top products to analyze  
      - Titles missing or empty  
      - Stop word list may need localization or expansion  
    - **Version:** 2  

---

#### 1.5 Pricing Strategy Formulation

- **Overview:**  
  Generates pricing strategies (competitive, value, premium) based on product pricing data and SEO keyword insights, including risk assessment.

- **Nodes Involved:**  
  - Pricing Strategy

- **Node Details:**

  - **Pricing Strategy (Code node)**  
    - **Type & Role:** JavaScript Code node creating pricing recommendations and competitive positioning.  
    - **Configuration:**  
      - Uses average market price and price distribution from product analysis.  
      - Defines price ranges and pros/cons for each strategy.  
      - Analyzes risk based on quality gaps and price competition.  
      - Lists prioritized action items for implementation.  
    - **Input/Output:** Inputs from Product Analyzer and Keyword Analyzer; outputs pricing strategy JSON to Report Generator.  
    - **Failures/Edge Cases:**  
      - Missing or zero average price  
      - No distribution data  
      - Inconsistent input data structures  
    - **Version:** 2  

---

#### 1.6 Reporting

- **Overview:**  
  Produces a comprehensive strategic report combining market overview, key insights, recommendations, phased action plans, and KPIs.

- **Nodes Involved:**  
  - Report Generator

- **Node Details:**

  - **Report Generator (Code node)**  
    - **Type & Role:** JavaScript Code node assembling final report JSON.  
    - **Configuration:**  
      - Aggregates inputs: product analysis, keyword analysis, pricing strategy.  
      - Evaluates market health, generates insights and recommendations by timeline (immediate, short-term, long-term).  
      - Defines KPIs for tracking performance.  
      - Creates phased action plan with priorities and timelines.  
    - **Input/Output:** Inputs from Product Analyzer, Keyword Analyzer, Pricing Strategy; outputs report JSON to Google Docs node.  
    - **Failures/Edge Cases:**  
      - Missing input data from previous nodes  
      - Date/time formatting issues  
      - Logic errors in market health calculation  
    - **Version:** 2  

---

#### 1.7 Google Docs Integration

- **Overview:**  
  Automates creation of a timestamped Google Document containing the generated report for sharing and archival.

- **Nodes Involved:**  
  - Create Google Doc

- **Node Details:**

  - **Create Google Doc**  
    - **Type & Role:** Google Docs node, creates a new document with a dynamic title.  
    - **Configuration:**  
      - Title uses expression `Amazon Market Analysis - {{$json.generated_at}}` for uniqueness.  
      - Requires OAuth2 Google API credentials with Docs API enabled.  
      - Outputs the new document URL or metadata.  
    - **Input/Output:** Receives report JSON; outputs Google Docs metadata.  
    - **Failures/Edge Cases:**  
      - OAuth2 token expiration or misconfiguration  
      - Insufficient Google Drive permissions  
      - API quota limits reached  
    - **Version:** 2  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                        | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                                                                                  |
|-------------------------|--------------------------------|-------------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Overview  | Sticky Note                    | High-level workflow description      |                          |                          | # 🎯 WORKFLOW OVERVIEW Purpose: Automated Amazon market intelligence system that runs daily to provide competitive insights ... Execution: Daily at 6:00 AM UTC                |
| Daily Schedule          | Schedule Trigger               | Triggers daily workflow start        |                          | Amazon Product Scraper    | 🎯 WORKFLOW OVERVIEW Esecuzione automatica giornaliera alle 6:00 Analizza mercato Amazon per: • Pricing intelligence • Competitor analysis • Keyword opportunities • Strategic insights |
| Amazon Product Scraper  | ScrapeGraphAI Node             | Scrapes Amazon product data          | Daily Schedule            | Product Analyzer          | 🔧 SCRAPING CONFIG ⚠️ IMPORTANTE: • Configurare credenziali ScrapeGraphAI • Modificare URL per categoria specifica • Monitorare rate limits Amazon • Aggiungere IP rotation se necessario |
| Product Analyzer        | Code Node                     | Analyzes scraped product data        | Amazon Product Scraper    | Keyword Analyzer          | 📊 CORE ANALYSIS Analizza automaticamente: ✓ Prezzi (min/max/media/distribuzione) ✓ Rating e qualità prodotti ✓ Top performers del mercato ✓ Gap di qualità e prezzo ✓ Opportunità competitive |
| Keyword Analyzer        | Code Node                     | Extracts keywords & SEO insights      | Product Analyzer          | Pricing Strategy          | 🔍 SEO & KEYWORD STRATEGY Keyword Extraction: - Analyzes top performer titles - Identifies high-frequency terms - Filters out stop words ... PPC campaign keywords             |
| Pricing Strategy        | Code Node                     | Generates pricing strategies          | Keyword Analyzer          | Report Generator          | 💰 PRICING STRATEGIES Three Strategic Approaches: Competitive, Value, Premium pricing ... Risk Assessment: Competition level analysis, Market saturation indicators, Quality gap opportunities |
| Report Generator        | Code Node                     | Creates comprehensive strategic report| Pricing Strategy          | Create Google Doc         | 📋 STRATEGIC REPORT Executive Summary completo • Piano d'azione per fasi temporali • KPI da monitorare quotidianamente • Insights strategici actionable • Raccomandazioni immediate e LT |
| Create Google Doc       | Google Docs                   | Creates Google Doc with report        | Report Generator          |                          | 🔐 GOOGLE DOCS INTEGRATION Setup Requirements: Google OAuth2 API credentials, Docs API enabled, Shared folder permissions ... Automation Benefits: Daily market intelligence, etc. |
| Sticky Note - Setup     | Sticky Note                   | Setup and credential requirements     |                          |                          | # ⚙️ SETUP REQUIREMENTS Required Credentials: 1. ScrapeGraphAI API key 2. Google OAuth2 (Docs API) ... Rate Limits: Amazon, ScrapeGraphAI                                   |
| Sticky Note - Analysis  | Sticky Note                   | Description of analysis outputs       |                          |                          | # 📊 ANALYSIS OUTPUTS Pricing Intelligence, Quality Analysis, Market Opportunities, Positioning recommendations                                                          |
| Sticky Note - SEO       | Sticky Note                   | SEO & keyword strategy explanation    |                          |                          | # 🔍 SEO & KEYWORD STRATEGY Keyword Extraction, Market Trends, Competitive Intelligence, Actionable Outputs                                                                |
| Sticky Note - Pricing   | Sticky Note                   | Pricing strategies explanation        |                          |                          | # 💰 PRICING STRATEGIES Three Strategic Approaches: Competitive, Value, Premium Pricing; Risk Assessment and Positioning                                                   |
| Sticky Note - Reporting | Sticky Note                   | Strategic reporting explanation       |                          |                          | # 📋 STRATEGIC REPORTING Executive Summary, Phased Action Plan, KPI Tracking, Output to Google Docs                                                                       |
| Sticky Note - Google Docs| Sticky Note                  | Google Docs integration notes         |                          |                          | # 🔐 GOOGLE DOCS INTEGRATION Setup, Report Features, Automation Benefits, Access                                                                                           |
| Sticky Note - Business Impact| Sticky Note              | Business benefits and next steps      |                          |                          | # 🚀 BUSINESS IMPACT Time Savings, Strategic Advantages, Success Metrics, Scalability, Next Steps                                                                          |
| Sticky Note - Considerations| Sticky Note               | Legal, data quality, maintenance      |                          |                          | # ⚠️ IMPORTANT CONSIDERATIONS Legal & Compliance, Data Quality, Maintenance, Security                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node named "Daily Schedule".**  
   - Set to cron expression `0 6 * * *` for daily 6:00 AM UTC trigger.

2. **Add a ScrapeGraphAI node named "Amazon Product Scraper".**  
   - Configure ScrapeGraphAI API credentials.  
   - Set website URL to `https://www.amazon.com/s?k=wireless+bluetooth+headphones`.  
   - User prompt: "Extract product data: title, price, rating, review_count, asin, prime_eligible, image_url. Return as JSON array."  
   - Connect "Daily Schedule" output to this node’s input.

3. **Add a Code node named "Product Analyzer".**  
   - Paste the JavaScript code that:  
     - Parses the product array, analyzes pricing stats (average, min, max, distribution), rating distribution, top products, market opportunities, and insights.  
   - Connect "Amazon Product Scraper" output to this node’s input.

4. **Add a Code node named "Keyword Analyzer".**  
   - Paste the JavaScript code that extracts keywords from top product titles, filters stop words, counts frequencies, identifies trends and recommendations.  
   - Connect "Product Analyzer" output to this node’s input.

5. **Add a Code node named "Pricing Strategy".**  
   - Paste the JavaScript code generating pricing strategies (competitive, value, premium), competitive positioning, risk assessment, and action items.  
   - Connect "Keyword Analyzer" output to this node’s input.

6. **Add a Code node named "Report Generator".**  
   - Paste the JavaScript code that aggregates data into a comprehensive report with executive summary, key insights, recommendations by timeline, phased action plans, and KPIs.  
   - Connect "Pricing Strategy" output to this node’s input.

7. **Add a Google Docs node named "Create Google Doc".**  
   - Configure Google OAuth2 credentials with Docs API enabled.  
   - Set document title expression to: `Amazon Market Analysis - {{$json.generated_at}}`.  
   - Connect "Report Generator" output to this node’s input.

8. **Add optional Sticky Note nodes to document each block (Setup, Overview, Analysis, SEO, Pricing, Reporting, Google Docs, Business Impact, Considerations).**  
   - These are for reference and do not affect workflow execution.

9. **Ensure credential setup:**  
   - ScrapeGraphAI API key credential configured and tested.  
   - Google OAuth2 credential with Docs API enabled and Drive folder permissions granted.

10. **Test the workflow:**  
    - Run manually first with limited product set.  
    - Verify output data at each step.  
    - Check Google Doc creation and content formatting.

11. **Schedule activation:**  
    - Enable the workflow to run automatically daily.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Respect Amazon's robots.txt and Terms of Service when scraping data to avoid legal issues and IP bans.                      | Sticky Note - Considerations                   |
| Monitor API rate limits for ScrapeGraphAI and implement IP rotation if scraping volume increases.                           | Sticky Note - Setup                             |
| Google Docs API must be enabled in Google Cloud Console with OAuth2 credentials properly configured and folder sharing set. | Sticky Note - Google Docs                       |
| Workflow saves 10+ hours of manual research weekly, enabling real-time competitive intelligence and strategic decision making.| Sticky Note - Business Impact                   |
| Use phased action plans for market entry, optimization, and expansion with clear KPIs to track success over time.           | Sticky Note - Reporting                         |
| For SEO, focus on long-tail keywords and voice search optimization to capture emerging market trends.                       | Sticky Note - SEO                               |
| Regularly validate and update scraping selectors to accommodate Amazon layout changes and maintain data quality.            | Sticky Note - Considerations                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies fully with content policies, contains no illegal or offensive material, and manipulates only legal and public data.

---