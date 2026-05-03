Real Estate Cold Call Scripts for Price Reduced FSBO Properties (Zillow Data)

https://n8nworkflows.xyz/workflows/real-estate-cold-call-scripts-for-price-reduced-fsbo-properties--zillow-data--3143


# Real Estate Cold Call Scripts for Price Reduced FSBO Properties (Zillow Data)

### 1. Workflow Overview

This workflow automates the identification, analysis, and outreach process for FSBO (For Sale By Owner) real estate properties on Zillow that have recently reduced their prices. It integrates Zillow market data, investment calculations, and AI-generated communication templates to streamline investor outreach efforts.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a form submission to start the process.
- **1.2 Property Criteria Setup:** Defines search parameters for Zillow API queries.
- **1.3 Zillow Data Retrieval and Processing:** Searches Zillow listings, splits results, and fetches RentZestimate data.
- **1.4 Market Overview and Analysis:** Retrieves market data, processes it, and generates historical market insights using AI.
- **1.5 Investment Metrics Calculation:** Calculates financial metrics and filters properties based on investment quality.
- **1.6 Communication Template Generation:** Uses AI to create personalized call scripts based on property and market data.
- **1.7 Data Storage and Status Management:** Uploads final data and scripts to Airtable for tracking and outreach.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow upon receiving a form submission, triggering the entire automation sequence.

- **Nodes Involved:**  
  - On form submission  
  - Execute Workflow

- **Node Details:**  
  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts the workflow when a user submits a form (likely a manual trigger or external input).  
    - *Configuration:* Default, listens for incoming form data.  
    - *Connections:* Outputs to Execute Workflow node.  
    - *Edge Cases:* Missing or malformed form data could cause downstream failures.  
  - **Execute Workflow**  
    - *Type:* Execute Workflow  
    - *Role:* Invokes the sub-workflow that sets property criteria and continues processing.  
    - *Configuration:* Calls the "FSBO Property Criteria Set" node next.  
    - *Connections:* Input from On form submission; output to FSBO Property Criteria Set node.  
    - *Edge Cases:* Workflow execution errors if sub-workflow is missing or misconfigured.

#### 2.2 Property Criteria Setup

- **Overview:**  
  Defines and sets the search parameters for Zillow API queries to filter FSBO properties with recent price reductions.

- **Nodes Involved:**  
  - FSBO Property Criteria Set

- **Node Details:**  
  - **FSBO Property Criteria Set**  
    - *Type:* Set  
    - *Role:* Prepares the search criteria such as listing type, location, price reduction filters, and other property filters.  
    - *Configuration:* Sets parameters like "For Sale By Owner," price reduction = true, excludes auctions, filters by days since price reduction, minimum bedrooms/sqft.  
    - *Connections:* Input from Execute Workflow; output to Zillow Search node.  
    - *Edge Cases:* Incorrect filter values may result in empty or irrelevant search results.

#### 2.3 Zillow Data Retrieval and Processing

- **Overview:**  
  Queries Zillow's Rapid API for properties matching criteria, splits the results for individual processing, and fetches RentZestimate data for each property.

- **Nodes Involved:**  
  - Zillow Search  
  - Split Out  
  - RentZestimate  
  - Investment Calculator

- **Node Details:**  
  - **Zillow Search**  
    - *Type:* HTTP Request  
    - *Role:* Calls Zillow API with search parameters to retrieve FSBO listings with price reductions.  
    - *Configuration:* Uses HTTP GET/POST with authentication to Zillow Rapid API; includes filters from previous node.  
    - *Connections:* Input from FSBO Property Criteria Set; output to Split Out.  
    - *Edge Cases:* API rate limits, authentication failures, empty responses.  
  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits the array of Zillow listings into individual items for sequential processing.  
    - *Configuration:* Default splitting of array items.  
    - *Connections:* Input from Zillow Search; output to RentZestimate.  
    - *Edge Cases:* Empty input array leads to no further processing.  
  - **RentZestimate**  
    - *Type:* HTTP Request  
    - *Role:* Fetches rental estimate data for each property to assist in investment calculations.  
    - *Configuration:* Calls Zillow or related API endpoint for RentZestimate data using property identifiers.  
    - *Connections:* Input from Split Out; output to Investment Calculator.  
    - *Edge Cases:* Missing RentZestimate data triggers fallback logic in investment calculator.  
  - **Investment Calculator**  
    - *Type:* Code  
    - *Role:* Calculates investment metrics including mortgage, expenses, cash flow, ROI, and filters for positive cash flow properties.  
    - *Configuration:* Custom JavaScript code implementing financial formulas and fallback rules (e.g., 1% rent rule).  
    - *Connections:* Input from RentZestimate; output to Call Script Generator.  
    - *Edge Cases:* Calculation errors if input data is incomplete or malformed.

#### 2.4 Market Overview and Analysis

- **Overview:**  
  Retrieves comprehensive market data, processes it through custom code, and generates AI-driven historical market summaries to inform investment strategy.

- **Nodes Involved:**  
  - FSBO (Execute Workflow Trigger)  
  - market_overview  
  - Edit Fields1  
  - Historical Market Results Indicator  
  - Historical Market Summary  
  - OpenAI Chat Model1

- **Node Details:**  
  - **FSBO**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Starts the market overview sub-workflow.  
    - *Connections:* Output to market_overview.  
  - **market_overview**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves market-level data such as inventory, pricing trends, days on market, and sale ratios.  
    - *Configuration:* Calls external API or internal endpoint with location parameters.  
    - *Connections:* Input from FSBO; output to Edit Fields1.  
    - *Edge Cases:* API failures or stale data.  
  - **Edit Fields1**  
    - *Type:* Set  
    - *Role:* Prepares and formats market data fields for further processing.  
    - *Connections:* Input from market_overview; output to Historical Market Results Indicator.  
  - **Historical Market Results Indicator**  
    - *Type:* Code  
    - *Role:* Applies custom logic to analyze historical market data and generate indicators like market cycle position and trend scores.  
    - *Connections:* Input from Edit Fields1; output to Historical Market Summary.  
    - *Edge Cases:* Logic errors or missing data.  
  - **Historical Market Summary**  
    - *Type:* Chain Summarization (LangChain)  
    - *Role:* Summarizes processed market data into a concise narrative for strategic guidance.  
    - *Connections:* Input from Historical Market Results Indicator; output to OpenAI Chat Model1.  
  - **OpenAI Chat Model1**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Uses OpenAI to generate refined market analysis text based on summarized data.  
    - *Connections:* Input from Historical Market Summary; output is final market intelligence text.  
    - *Edge Cases:* API quota limits, prompt failures.

#### 2.5 Communication Template Generation

- **Overview:**  
  Generates personalized call scripts combining property details, market intelligence, and investment metrics using OpenAI, then stores them in Airtable.

- **Nodes Involved:**  
  - Call Script Generator  
  - Call Script Database Airtable

- **Node Details:**  
  - **Call Script Generator**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Creates customized outreach scripts referencing price reductions, market conditions, and investment offers.  
    - *Configuration:* Uses property and market data as prompt context; generates persuasive text.  
    - *Connections:* Input from Investment Calculator; output to Call Script Database Airtable.  
    - *Edge Cases:* AI output variability, API limits.  
  - **Call Script Database Airtable**  
    - *Type:* Airtable  
    - *Role:* Uploads generated scripts and property data to Airtable for tracking and outreach.  
    - *Configuration:* Uses Airtable API credentials; writes to specified base and table with fields like Status and Follow-up Date.  
    - *Connections:* Input from Call Script Generator.  
    - *Edge Cases:* API authentication errors, rate limits, data schema mismatches.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                          | Input Node(s)                   | Output Node(s)                   | Sticky Note                                      |
|-------------------------------|----------------------------------|----------------------------------------|--------------------------------|---------------------------------|-------------------------------------------------|
| On form submission             | Form Trigger                     | Workflow start trigger                  | -                              | Execute Workflow                |                                                 |
| Execute Workflow              | Execute Workflow                 | Invokes sub-workflow for criteria setup| On form submission             | FSBO Property Criteria Set       |                                                 |
| FSBO Property Criteria Set     | Set                             | Defines Zillow search filters           | Execute Workflow               | Zillow Search                   |                                                 |
| Zillow Search                 | HTTP Request                    | Queries Zillow API for FSBO listings    | FSBO Property Criteria Set     | Split Out                      |                                                 |
| Split Out                    | Split Out                       | Splits Zillow listings array             | Zillow Search                 | RentZestimate                  |                                                 |
| RentZestimate                 | HTTP Request                    | Fetches rental estimate data             | Split Out                    | Investment Calculator          |                                                 |
| Investment Calculator         | Code                            | Calculates investment metrics            | RentZestimate                | Call Script Generator          |                                                 |
| Call Script Generator         | OpenAI (LangChain)              | Generates personalized call scripts     | Investment Calculator         | Call Script Database Airtable  |                                                 |
| Call Script Database Airtable | Airtable                        | Stores scripts and property data        | Call Script Generator         | -                             |                                                 |
| FSBO                         | Execute Workflow Trigger         | Starts market overview sub-workflow     | Execute Workflow (indirect)   | market_overview                |                                                 |
| market_overview              | HTTP Request                    | Retrieves market data                     | FSBO                         | Edit Fields1                  |                                                 |
| Edit Fields1                 | Set                             | Formats market data                       | market_overview              | Historical Market Results Indicator |                                                 |
| Historical Market Results Indicator | Code                      | Analyzes historical market data          | Edit Fields1                 | Historical Market Summary      |                                                 |
| Historical Market Summary    | Chain Summarization (LangChain) | Summarizes market data                    | Historical Market Results Indicator | OpenAI Chat Model1          |                                                 |
| OpenAI Chat Model1           | LangChain OpenAI Chat Model      | Generates refined market analysis text  | Historical Market Summary    | -                             |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Form Trigger** node named "On form submission" to start the workflow on form input.

2. **Execute Sub-Workflow:**  
   - Add an **Execute Workflow** node named "Execute Workflow" connected from "On form submission."  
   - This node will invoke the property criteria setup and subsequent processing.

3. **Set Property Search Criteria:**  
   - Add a **Set** node named "FSBO Property Criteria Set."  
   - Configure fields to specify:  
     - Listing type: "For Sale By Owner"  
     - Price reduction: true  
     - Exclude auctions  
     - Filters for minimum bedrooms, square footage, and max days since price reduction (e.g., 14 days).

4. **Zillow API Search:**  
   - Add an **HTTP Request** node named "Zillow Search."  
   - Configure it to call Zillow Rapid API with authentication and the criteria from the previous node.  
   - Use GET or POST as required by Zillow API.  
   - Connect "FSBO Property Criteria Set" output to this node.

5. **Split Zillow Results:**  
   - Add a **Split Out** node named "Split Out" to split the array of listings into individual items.  
   - Connect from "Zillow Search."

6. **Fetch RentZestimate Data:**  
   - Add an **HTTP Request** node named "RentZestimate."  
   - Configure it to call Zillow or related API to get rental estimates for each property.  
   - Connect from "Split Out."

7. **Calculate Investment Metrics:**  
   - Add a **Code** node named "Investment Calculator."  
   - Implement calculations for:  
     - Purchase price, down payment, closing costs  
     - Loan amount, mortgage payment  
     - Property tax, insurance, maintenance, vacancy allowance  
     - RentZestimate fallback to 1% rule if missing  
     - Cash flow, ROI, break-even timeline  
     - Filter for positive cash flow properties only  
   - Connect from "RentZestimate."

8. **Generate Call Scripts:**  
   - Add an **OpenAI (LangChain)** node named "Call Script Generator."  
   - Configure with OpenAI API credentials.  
   - Use property details, market data, and investment metrics as prompt context to generate personalized scripts.  
   - Connect from "Investment Calculator."

9. **Store Scripts in Airtable:**  
   - Add an **Airtable** node named "Call Script Database Airtable."  
   - Configure with Airtable API key and base/table details.  
   - Map fields including property info, communication template, status, and follow-up date.  
   - Connect from "Call Script Generator."

10. **Market Overview Sub-Workflow:**  
    - Add an **Execute Workflow Trigger** node named "FSBO" to start market data retrieval.  
    - Connect from "Execute Workflow" or appropriate node to trigger market analysis.

11. **Retrieve Market Data:**  
    - Add an **HTTP Request** node named "market_overview."  
    - Configure to fetch market indicators like inventory, pricing trends, days on market, sale ratios.  
    - Connect from "FSBO."

12. **Format Market Data:**  
    - Add a **Set** node named "Edit Fields1" to prepare market data fields.  
    - Connect from "market_overview."

13. **Analyze Historical Market Data:**  
    - Add a **Code** node named "Historical Market Results Indicator."  
    - Implement logic to score market trends, identify market cycle positions, and generate indicators.  
    - Connect from "Edit Fields1."

14. **Summarize Market Data:**  
    - Add a **Chain Summarization (LangChain)** node named "Historical Market Summary."  
    - Configure to summarize the processed market data into a narrative.  
    - Connect from "Historical Market Results Indicator."

15. **Generate AI Market Analysis Text:**  
    - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model1."  
    - Configure with OpenAI API credentials.  
    - Connect from "Historical Market Summary."

16. **Connect Market Analysis Output:**  
    - Integrate the output of "OpenAI Chat Model1" back into the investment or communication template generation nodes as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                           |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Step-by-step video guide for this workflow setup                                              | https://youtu.be/IqVw9CIL254?si=MKE5UY4rD0TOMLPg          |
| Requires Airtable account with Personal Access Token for API integration                      | Airtable API documentation                                 |
| Zillow Rapid API access needed for property and RentZestimate data                            | Zillow Rapid API documentation                             |
| OpenAI API key required for AI-driven market summaries and communication template generation | OpenAI API documentation                                   |
| Workflow designed to filter and focus on positive cash flow properties only                   | Investment filtering logic embedded in Investment Calculator node |
| Adjust calculation parameters to tune offer prices and risk assessments                       | Editable in Investment Calculator code node               |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the "Real Estate Cold Call Scripts for Price Reduced FSBO Properties" workflow, enabling efficient automation of real estate investment outreach.