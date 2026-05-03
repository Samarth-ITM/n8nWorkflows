Carbon Footprint Tracker with ScrapeGraphAI Analysis and Google Drive ESG Reports

https://n8nworkflows.xyz/workflows/carbon-footprint-tracker-with-scrapegraphai-analysis-and-google-drive-esg-reports-6643


# Carbon Footprint Tracker with ScrapeGraphAI Analysis and Google Drive ESG Reports

### 1. Workflow Overview

This workflow, titled **"Carbon Footprint Tracker with ScrapeGraphAI Analysis and Google Drive ESG Reports"**, automates the process of daily carbon footprint calculation, reduction opportunity analysis, dashboard preparation, ESG report generation, and report storage for an organization. Targeted at sustainability teams and environmental analysts, it integrates real-time web scraping, computational logic, and cloud storage to facilitate continuous ESG (Environmental, Social, and Governance) monitoring and reporting.

The workflow logically divides into these functional blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a fixed time.
- **1.2 Data Collection:** Parallel scraping of energy consumption and transportation emission data from authoritative web sources via ScrapeGraphAI nodes.
- **1.3 Carbon Footprint Calculation:** Computes a comprehensive carbon footprint using organizational baseline data and scraped emission factors.
- **1.4 Reduction Opportunity Analysis:** Identifies actionable opportunities to reduce emissions with investment and ROI details.
- **1.5 Dashboard Data Preparation:** Formats the analysis results into structured data for visualization dashboards.
- **1.6 ESG Report Generation:** Produces a detailed markdown-formatted ESG report encapsulating analysis, opportunities, and strategic recommendations.
- **1.7 Report Storage:** Creates a dedicated Google Drive folder and uploads the generated report for team access and historical record.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
Triggers the entire workflow automatically every day at 8:00 AM to enable consistent daily monitoring and data updates.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger (built-in n8n node)  
    - Configuration: Cron expression `0 8 * * *` to run daily at 8:00 AM local time  
    - Input: N/A (trigger node)  
    - Output: Triggers parallel downstream scraping nodes  
    - Edge Cases: Cron misconfiguration, timezone mismatches, or workflow disablement can cause missed runs  
    - Sticky Note: "Step 1: Daily Trigger ⏰" explains scheduling options and purpose

#### 1.2 Data Collection

- **Overview:**  
Scrapes current energy consumption and transportation emission data from EPA and FuelEconomy.gov websites using AI-powered scraping nodes, running in parallel.

- **Nodes Involved:**  
  - Energy Data Scraper  
  - Transport Data Scraper

- **Node Details:**  
  - **Energy Data Scraper**  
    - Type: ScrapeGraphAI Node  
    - Configuration: Uses a prompt to extract structured energy consumption and carbon emission factors from EPA's Greenhouse Gas Equivalencies Calculator webpage  
    - Credentials: Requires valid ScrapeGraphAI API credentials  
    - Output: JSON object conforming to schema with energy type, consumption, units, carbon factor, and source metadata  
    - Input: Triggered by Schedule Trigger  
    - Edge Cases: Website layout changes, API auth failure, network errors, or incomplete data extraction  
  - **Transport Data Scraper**  
    - Type: ScrapeGraphAI Node  
    - Configuration: Extracts vehicle emission factors and fuel efficiency data from FuelEconomy.gov  
    - Credentials: Same as above  
    - Output: JSON object with vehicle type, fuel type, MPG, CO2 emissions per gallon and mile, source, and category  
    - Input: Triggered in parallel by Schedule Trigger  
    - Edge Cases: Same as Energy Data Scraper, plus possible data inconsistencies or schema mismatches  
  - Sticky Note: "Step 2: Data Collection 🌐" details sources and data types gathered

#### 1.3 Carbon Footprint Calculation

- **Overview:**  
Aggregates scraped data and organizational baseline metrics to compute total carbon emissions across Scope 1 (direct), Scope 2 (indirect electricity), and Scope 3 (other indirect) categories, including per-employee calculations.

- **Nodes Involved:**  
  - Footprint Calculator

- **Node Details:**  
  - **Footprint Calculator**  
    - Type: Code Node (JavaScript)  
    - Configuration: Custom JS code combines input JSON from Energy Data Scraper (input 0) and Transport Data Scraper (input 1) with hardcoded organization data (electricity use, natural gas, fleet miles, employee commute, air travel, number of employees)  
    - Key Logic: Calculates emissions using emission factors and consumption data, totals emissions by scope, computes per-employee values  
    - Outputs: Structured JSON with timestamp, total emissions (lbs and tons CO2e), scope breakdown, per-employee emissions, and baseline data  
    - Input: Receives data arrays from both scrapers in parallel  
    - Edge Cases: Missing input data, division by zero if employees = 0, calculation errors due to unexpected data formats  
    - Sticky Note: "Step 3: Footprint Calculator 🧮" explains scope breakdown and output details

#### 1.4 Reduction Opportunity Analysis

- **Overview:**  
Analyzes carbon footprint data to identify prioritized reduction opportunities with estimated emissions reductions, investment needs, payback periods, and implementation effort.

- **Nodes Involved:**  
  - Reduction Opportunity Finder

- **Node Details:**  
  - **Reduction Opportunity Finder**  
    - Type: Code Node (JavaScript)  
    - Configuration: Analyzes footprint totals and breakdowns to conditionally recommend energy efficiency upgrades (LED lighting, solar panels), transportation policies (remote work, EV fleet), and office operations improvements (smart HVAC)  
    - Outputs: JSON containing current footprint, list of opportunities with metadata, total potential reduction in tons, percentage reduction, and analysis timestamp  
    - Input: Receives footprint data from Footprint Calculator  
    - Edge Cases: No opportunities identified if inputs are missing or thresholds not met, output formatting errors  
    - Sticky Note: "Step 4: Opportunity Analysis 🎯" describes categories and analysis details

#### 1.5 Dashboard Data Preparation

- **Overview:**  
Formats the reduction analysis into structured data for use in a sustainability dashboard, including KPIs, emission scopes, monthly trends, and top opportunities.

- **Nodes Involved:**  
  - Sustainability Dashboard

- **Node Details:**  
  - **Sustainability Dashboard**  
    - Type: Code Node (JavaScript)  
    - Configuration: Constructs multiple objects for dashboard UI elements: KPI cards with values, trends, and statuses; pie chart data for emission scopes; simulated monthly emission trends; prioritized top opportunities with impact scores  
    - Outputs: JSON including dashboard data object and raw analysis for downstream use  
    - Input: Receives reduction analysis data  
    - Edge Cases: Missing or malformed input data, unexpected sorting errors  
    - Sticky Note: "Step 5: Dashboard Preparation 📊" outlines dashboard components and outputs

#### 1.6 ESG Report Generation

- **Overview:**  
Generates a comprehensive ESG report in markdown format compiling the analysis, emission breakdown, reduction opportunities, strategic recommendations, and financial impacts.

- **Nodes Involved:**  
  - ESG Report Generator

- **Node Details:**  
  - **ESG Report Generator**  
    - Type: Code Node (JavaScript)  
    - Configuration: Uses dashboard and raw analysis data to build a multi-section report including executive summary, detailed emissions breakdown (by scope and source), opportunity listings, strategic recommendations split by timeframe, compliance benchmarking, and next steps  
    - Outputs: JSON with full markdown report text, report metadata (date, type), key metrics summary, and filename with ISO date suffix  
    - Input: Receives dashboard data from Sustainability Dashboard  
    - Edge Cases: String interpolation failures, date formatting issues, excessively long texts  
    - Sticky Note: "Step 6: ESG Report Generation 📋" details report content and compliance features

#### 1.7 Report Storage

- **Overview:**  
Creates a dedicated folder on Google Drive and uploads the generated markdown ESG report for easy team access, historical archiving, and version control.

- **Nodes Involved:**  
  - Create Reports Folder  
  - Save Report to Drive

- **Node Details:**  
  - **Create Reports Folder**  
    - Type: Google Drive Node  
    - Operation: Create folder named "ESG_Reports" in root or configured drive  
    - Credentials: Requires Google Drive OAuth2 credentials with folder creation permissions  
    - Output: Folder metadata including folder ID  
    - Input: Receives trigger from ESG Report Generator  
    - Edge Cases: Folder already exists, permission errors, API quota limits  
  - **Save Report to Drive**  
    - Type: Google Drive Node  
    - Operation: Upload a file with the report text content, named dynamically per generated filename, saved inside the created folder  
    - Input: Uses folder ID from "Create Reports Folder" and report text from ESG Report Generator  
    - Edge Cases: File upload failures, permission issues, naming conflicts  
    - Sticky Note: "Step 7: Report Storage 💾" describes folder use, versioning, and collaboration features

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                   | Input Node(s)               | Output Node(s)                       | Sticky Note                                                                                         |
|-------------------------|---------------------------|---------------------------------|-----------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger          | Initiates workflow daily         | N/A                         | Energy Data Scraper, Transport Data Scraper | Step 1: Daily Trigger ⏰ Explains scheduling details                                               |
| Energy Data Scraper     | ScrapeGraphAI             | Scrapes energy consumption data | Schedule Trigger            | Footprint Calculator                 | Step 2: Data Collection 🌐 Details data sources and scrape targets                                |
| Transport Data Scraper  | ScrapeGraphAI             | Scrapes transportation emission data | Schedule Trigger          | Footprint Calculator                 | Step 2: Data Collection 🌐 (shared with Energy Data Scraper)                                      |
| Footprint Calculator    | Code                      | Calculates carbon footprint      | Energy Data Scraper, Transport Data Scraper | Reduction Opportunity Finder          | Step 3: Footprint Calculator 🧮 Explains scope calculation and output                              |
| Reduction Opportunity Finder | Code                 | Identifies emission reduction opportunities | Footprint Calculator     | Sustainability Dashboard             | Step 4: Opportunity Analysis 🎯 Details opportunity categories and ROI                            |
| Sustainability Dashboard| Code                      | Prepares dashboard data          | Reduction Opportunity Finder | ESG Report Generator                 | Step 5: Dashboard Preparation 📊 Describes dashboard KPIs and chart data                          |
| ESG Report Generator    | Code                      | Generates markdown ESG report    | Sustainability Dashboard    | Create Reports Folder                | Step 6: ESG Report Generation 📋 Describes report content and compliance features                 |
| Create Reports Folder   | Google Drive              | Creates folder for reports       | ESG Report Generator        | Save Report to Drive                 | Step 7: Report Storage 💾 Describes folder creation and versioning                                |
| Save Report to Drive    | Google Drive              | Uploads report file to Drive     | Create Reports Folder       | None                                | Step 7: Report Storage 💾 (shared with Create Reports Folder)                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set Cron Expression: `0 8 * * *` (daily at 8:00 AM)  
   - No credentials needed  
   - Position roughly at the top-left of your canvas

2. **Create Energy Data Scraper Node**  
   - Type: ScrapeGraphAI  
   - Configure credentials with your ScrapeGraphAI API key  
   - Set Website URL: `https://www.epa.gov/energy/greenhouse-gas-equivalencies-calculator`  
   - User Prompt: Extract and format energy consumption data into JSON schema:  
     ```
     { "energy_type": "electricity", "consumption_value": "...", "unit": "kWh", "carbon_factor": "...", "emission_unit": "lbs CO2/kWh", "source": "EPA", "last_updated": "YYYY-MM-DD" }
     ```  
   - Connect input from Schedule Trigger

3. **Create Transport Data Scraper Node**  
   - Type: ScrapeGraphAI  
   - Use same ScrapeGraphAI credentials  
   - Set Website URL: `https://www.fueleconomy.gov/feg/co2.jsp`  
   - User Prompt: Extract transportation emission factors and fuel efficiency data as JSON:  
     ```
     { "vehicle_type": "passenger_car", "fuel_type": "gasoline", "mpg": "...", "co2_per_gallon": "...", "co2_per_mile": "...", "unit": "lbs CO2", "source": "EPA", "category": "transport" }
     ```  
   - Connect input from Schedule Trigger (in parallel with Energy Data Scraper)

4. **Create Footprint Calculator Node**  
   - Type: Code (JavaScript)  
   - Connect inputs from both Energy Data Scraper (index 0) and Transport Data Scraper (index 1)  
   - Paste the provided JS code that:  
     - Defines baseline org data (electricity, natural gas, fleet miles, etc.)  
     - Calculates emissions per scope and total footprint  
     - Outputs structured JSON with emissions and breakdown  
   - No special credentials required

5. **Create Reduction Opportunity Finder Node**  
   - Type: Code (JavaScript)  
   - Connect input from Footprint Calculator  
   - Paste provided JS code that:  
     - Analyzes footprint data to identify emission reduction opportunities  
     - Includes investment cost, payback, priority, and effort  
     - Outputs summary with potential reductions

6. **Create Sustainability Dashboard Node**  
   - Type: Code (JavaScript)  
   - Connect input from Reduction Opportunity Finder  
   - Paste provided JS code to format KPIs, scope charts, monthly trends, and top opportunities for dashboard visualization

7. **Create ESG Report Generator Node**  
   - Type: Code (JavaScript)  
   - Connect input from Sustainability Dashboard  
   - Paste provided JS code that composes a markdown-formatted ESG report with:  
     - Executive summary  
     - Emissions breakdown  
     - Opportunities and recommendations  
     - Compliance and next steps  
   - Outputs full report text and metadata

8. **Create Create Reports Folder Node**  
   - Type: Google Drive  
   - Operation: Create Folder  
   - Name: `ESG_Reports`  
   - Authenticate via Google Drive OAuth2 with folder creation permissions  
   - Connect input from ESG Report Generator

9. **Create Save Report to Drive Node**  
   - Type: Google Drive  
   - Operation: Upload File  
   - File name: Use expression to pull filename from ESG Report Generator output: `={{ $json.file_name }}`  
   - File content: Use expression `={{ $json.report_text }}`  
   - Set Parent folder to the ID output from Create Reports Folder node  
   - Use same Google Drive OAuth2 credentials  
   - Connect input from Create Reports Folder

10. **Validate and Test Workflow**  
    - Ensure all nodes have proper credentials and connections  
    - Run manually to verify outputs at each stage  
    - Adjust org baseline data and prompts as needed for your context

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow maintains daily automated carbon footprint tracking, enabling real-time ESG compliance and sustainability monitoring. | Overall Workflow Purpose                                  |
| Data sources include EPA’s Greenhouse Gas Equivalencies Calculator and FuelEconomy.gov for authoritative emission factors.  | ScrapeGraphAI Data Sources                                |
| Reports are generated in markdown format for easy readability and can be integrated with various documentation or wiki systems. | ESG Report Generator Output Format                        |
| Google Drive integration ensures centralized, accessible, and version-controlled report storage for sustainability teams.  | Report Storage and Collaboration                          |
| The workflow's modular design allows for easy extension, including adding manual triggers or alternative data sources.       | Workflow Design Flexibility                               |
| Sticky notes within the workflow provide detailed contextual guidance for each functional block.                            | Internal Documentation Support                            |
| For API credentials setup, refer to n8n official documentation for ScrapeGraphAI and Google Drive OAuth2 integrations.       | https://docs.n8n.io/nodes/credentials/                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. All processing complies strictly with current content policies and contains no illegal, offensive, or protected material. All handled data is lawful and publicly accessible.