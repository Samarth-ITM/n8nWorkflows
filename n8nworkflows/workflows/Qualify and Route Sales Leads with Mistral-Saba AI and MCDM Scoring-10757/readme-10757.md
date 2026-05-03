Qualify and Route Sales Leads with Mistral-Saba AI and MCDM Scoring

https://n8nworkflows.xyz/workflows/qualify-and-route-sales-leads-with-mistral-saba-ai-and-mcdm-scoring-10757


# Qualify and Route Sales Leads with Mistral-Saba AI and MCDM Scoring

### 1. Workflow Overview

This workflow automates the qualification and routing of sales leads using AI-powered multi-criteria decision making (MCDM) and natural language processing. It targets sales operations aiming to efficiently score, segment, and assign leads to appropriate sales teams (Enterprise, Mid-Market, SMB, or Nurture campaigns) based on firmographic, behavioral, and transactional data inputs. The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger and Configuration Setup:** Initiates the process on a scheduled interval and sets API endpoints and threshold parameters.
- **1.2 Data Acquisition:** Fetches demographic, behavioral, and transactional lead data from external APIs.
- **1.3 Data Aggregation and Scoring:** Merges data sources and applies an AHP-TOPSIS MCDM algorithm to score and rank leads.
- **1.4 AI-Based Lead Qualification:** Uses a LangChain AI agent with Mistral-Saba model to analyze scored leads and generate qualification insights.
- **1.5 Lead Score Preparation and Routing:** Prepares lead scores, assigns tiers, and routes leads to specific sales teams or nurture campaigns based on thresholds.
- **1.6 CRM Update and KPI Calculation:** Updates CRM systems with lead scores and qualification metadata; calculates and logs performance KPIs and analytics for continuous monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Configuration Setup

- **Overview:**  
Starts the workflow at regular intervals and initializes all configuration parameters, including API endpoints and scoring thresholds.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Workflow Configuration  
  - Sticky Notes (for documentation and setup instructions)

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Purpose: Triggers workflow every hour (configurable)  
    - Configuration: Interval set to 1 hour  
    - Inputs: None (trigger node)  
    - Outputs: Workflow Configuration node  
    - Edge Cases: Misconfigured intervals may cause unexpected runs or no runs  
  - **Workflow Configuration**  
    - Type: Set node  
    - Purpose: Sets string values for API URLs (demographic, behavioral, transactional, CRM, analytics) and numeric thresholds for lead scoring tiers (Enterprise, Mid-Market, SMB) plus AHP weights  
    - Configuration: Placeholder strings for URLs; numeric thresholds 85, 70, 50; AHP weights object for MCDM criteria  
    - Inputs: From Schedule Trigger  
    - Outputs: Fetch Demographic, Behavioral, Transactional Data nodes  
    - Edge Cases: Missing or incorrect API endpoints would cause HTTP request failures  
  - **Sticky Notes**  
    - Provide high-level workflow explanation, setup steps, prerequisites, use cases, customization options, and benefits  
    - Not part of execution flow but essential for user guidance

---

#### 2.2 Data Acquisition

- **Overview:**  
Fetches lead data from external APIs representing different data domains: demographic, behavioral, and transactional.

- **Nodes Involved:**  
  - Fetch Demographic Data  
  - Fetch Behavioral Data  
  - Fetch Transactional Data  
  - Sticky Notes (explanations for data fetching and parsing)

- **Node Details:**  
  - **Fetch Demographic Data**  
    - Type: HTTP Request  
    - Purpose: Retrieves demographic lead data from configured API endpoint  
    - Configuration: URL taken from Workflow Configuration; Content-Type header set to application/json  
    - Inputs: Workflow Configuration node  
    - Outputs: Merge Lead Data Sources  
    - Edge Cases: API unavailability, authentication errors, malformed responses  
  - **Fetch Behavioral Data**  
    - Same as above, but fetches behavioral data  
  - **Fetch Transactional Data**  
    - Same as above, but fetches transactional data  
  - **Sticky Notes**  
    - Describe the importance of multi-source data and the need for normalization for downstream AI and scoring

---

#### 2.3 Data Aggregation and Scoring

- **Overview:**  
Combines data from all sources into a unified lead profile and applies an advanced MCDM scoring algorithm (AHP-TOPSIS) to rank leads based on multiple weighted criteria.

- **Nodes Involved:**  
  - Merge Lead Data Sources  
  - MCDM Scoring Engine (AHP-TOPSIS)  
  - Sticky Notes (MCDM route explanation)

- **Node Details:**  
  - **Merge Lead Data Sources**  
    - Type: Aggregate  
    - Purpose: Aggregates all incoming lead data items into a single array under "leadData" field  
    - Inputs: Fetch Demographic, Behavioral, Transactional Data  
    - Outputs: MCDM Scoring Engine  
    - Edge Cases: Empty or inconsistent data arrays could affect scoring accuracy  
  - **MCDM Scoring Engine (AHP-TOPSIS)**  
    - Type: Code node (JavaScript)  
    - Purpose: Implements AHP-TOPSIS multi-criteria decision making to score and rank leads  
    - Key Logic:  
      - Extracts criteria values (companySize, budget, engagement, fitScore, urgency)  
      - Normalizes criteria values using vector normalization  
      - Applies configured weights  
      - Computes ideal best/worst values  
      - Calculates separation measures  
      - Computes closeness coefficient as MCDM score (range ~0-1)  
      - Ranks leads descending by score and assigns rank and percentile  
    - Inputs: Aggregated lead data  
    - Outputs: AI Lead Qualification Agent  
    - Edge Cases: Missing data fields default to zero; division by zero handled; malformed input JSON could cause runtime errors  
    - Version-specific: Requires n8n supporting JavaScript execution with date/time functions  
  - **Sticky Note**  
    - Explains routing logic based on MCDM scoring and the rationale behind lead assignment

---

#### 2.4 AI-Based Lead Qualification

- **Overview:**  
Uses a LangChain AI agent with the Mistral-Saba model to analyze scored leads, enrich them with insights, and provide actionable qualification assessments for sales teams.

- **Nodes Involved:**  
  - AI Lead Qualification Agent (LangChain agent)  
  - Lead Enrichment Tool (LangChain toolCode node)  
  - OpenRouter Chat Model (LangChain LM)  
  - Sticky Notes (AI scoring benefits and specifications)

- **Node Details:**  
  - **AI Lead Qualification Agent**  
    - Type: LangChain Agent node  
    - Purpose: Analyzes merged lead data with MCDM scores and generates qualification insights including strengths, objections, and recommendations  
    - Configuration: Uses text prompt defining expert lead qualification analyst role; calls Lead Enrichment Tool for additional data enrichment  
    - Inputs: MCDM Scoring Engine output, OpenRouter Chat Model, Lead Enrichment Tool  
    - Outputs: Prepare Lead Scores node  
    - Edge Cases: API call limits, model unavailability, prompt parsing errors  
  - **Lead Enrichment Tool**  
    - Type: LangChain toolCode (JavaScript)  
    - Purpose: Parses lead data input, interprets MCDM scores, assigns quality tiers (Enterprise, Mid-Market, SMB, Nurture), adds qualification insights and recommended actions  
    - Logic: Uses conditional checks on scores and behavioral data to enrich lead profiles  
    - Input: Query string from AI Lead Qualification Agent  
    - Output: JSON string of enriched lead profile  
  - **OpenRouter Chat Model**  
    - Type: LangChain Language Model node  
    - Purpose: Provides Mistral-Saba AI language model support for the AI Lead Qualification Agent  
    - Credentials: Requires OpenRouter API credentials  
    - Edge Cases: API rate limits, credential expiry  
  - **Sticky Notes**  
    - Document AI scoring rationale and benefits for accurate lead qualification

---

#### 2.5 Lead Score Preparation and Routing

- **Overview:**  
Prepares final lead score fields, assigns the lead quality tier, and routes leads to the appropriate sales teams or nurture campaigns based on configurable thresholds.

- **Nodes Involved:**  
  - Prepare Lead Scores (Set node)  
  - Route by Lead Quality (Switch node)  
  - Assign to Enterprise Sales Team (HTTP Request)  
  - Assign to Mid-Market Team (HTTP Request)  
  - Assign to SMB Team (HTTP Request)  
  - Send to Nurture Campaign (HTTP Request)  
  - Collect Routing Results (Aggregate)  
  - Sticky Notes (routing and team assignment explanations)

- **Node Details:**  
  - **Prepare Lead Scores**  
    - Type: Set node  
    - Purpose: Maps AI output fields to standardized variables: leadScore, leadTier, aiInsights, routingDecision  
    - Inputs: AI Lead Qualification Agent output  
    - Outputs: Route by Lead Quality node  
  - **Route by Lead Quality**  
    - Type: Switch node  
    - Purpose: Routes leads by comparing leadScore to thresholds defined in Workflow Configuration  
    - Conditions:  
      - Enterprise: score >= enterpriseThreshold (default 85)  
      - Mid-Market: score >= midMarketThreshold (default 70)  
      - SMB: score >= smbThreshold (default 50)  
      - Nurture: score < smbThreshold  
    - Outputs: Corresponding assignment HTTP requests  
  - **Assign to Enterprise Sales Team**  
    - Type: HTTP Request  
    - Purpose: Sends POST request to Enterprise Sales Team API with lead details, score, tier, and AI insights  
    - Edge Cases: API endpoint must be configured; handle HTTP errors  
  - **Assign to Mid-Market Team**  
    - Same as above, with Mid-Market API endpoint  
  - **Assign to SMB Team**  
    - Same as above, with SMB API endpoint  
  - **Send to Nurture Campaign**  
    - Type: HTTP Request  
    - Purpose: Sends lead details to nurture campaign API for low priority leads  
    - Payload includes lead email, name, score, qualification status, timestamp  
  - **Collect Routing Results**  
    - Type: Aggregate node  
    - Purpose: Aggregates all routing responses for subsequent CRM update  
  - **Sticky Notes**  
    - Explain routing decision logic and benefits of specialized team assignment for sales efficiency

---

#### 2.6 CRM Update and KPI Calculation

- **Overview:**  
Updates CRM with lead scores, routing decisions, and AI insights; calculates performance KPIs for monitoring and logs them to an analytics dashboard.

- **Nodes Involved:**  
  - Update CRM with Lead Scores (HTTP Request)  
  - Calculate Performance KPIs (Code node)  
  - Log KPIs to Analytics Dashboard (HTTP Request)  
  - Sticky Notes (CRM sync and KPI dashboard explanations)

- **Node Details:**  
  - **Update CRM with Lead Scores**  
    - Type: HTTP Request  
    - Purpose: Sends POST request to CRM API with lead data, scores, routing, AI insights, and timestamps  
    - Uses Workflow Configuration CRM API URL  
    - Edge Cases: CRM API failures, data mismatch, authentication errors  
  - **Calculate Performance KPIs**  
    - Type: Code node (JavaScript)  
    - Purpose: Computes metrics such as lead counts per tier, average scores, conversion rates, model accuracy, response and processing times  
    - Inputs: Aggregated updated CRM data  
    - Outputs: Log KPIs to Analytics Dashboard  
    - Edge Cases: Missing or inconsistent data, division by zero handled gracefully  
  - **Log KPIs to Analytics Dashboard**  
    - Type: HTTP Request  
    - Purpose: Sends KPI data to analytics dashboard API endpoint for real-time monitoring  
    - Uses Workflow Configuration analytics API URL  
    - Edge Cases: API availability, data schema mismatch  
  - **Sticky Notes**  
    - Describe rationale for syncing metadata and generating performance dashboards for strategic insights

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                                  | Input Node(s)                          | Output Node(s)                            | Sticky Note                                                                                                    |
|--------------------------------|---------------------------------|-------------------------------------------------|---------------------------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                 | Initiates workflow on schedule                    | None                                  | Workflow Configuration                    | ## Schedule Lead Processing: Triggers workflow at defined intervals to process batches of incoming leads.      |
| Workflow Configuration         | Set                             | Sets API URLs, thresholds, and AHP weights       | Schedule Trigger                     | Fetch Demographic Data, Fetch Behavioral Data, Fetch Transactional Data | ## Setup Steps: Define schedule, configure APIs, set routing rules, connect AI and CRM, link dashboards.       |
| Fetch Demographic Data         | HTTP Request                    | Retrieves demographic lead data                    | Workflow Configuration               | Merge Lead Data Sources                    | ## Fetch Behavioral Data: Retrieves multi-source lead info for rich intent signals.                            |
| Fetch Behavioral Data          | HTTP Request                    | Retrieves behavioral lead data                     | Workflow Configuration               | Merge Lead Data Sources                    | Same as above                                                                                                  |
| Fetch Transactional Data       | HTTP Request                    | Retrieves transactional lead data                  | Workflow Configuration               | Merge Lead Data Sources                    | Same as above                                                                                                  |
| Merge Lead Data Sources        | Aggregate                      | Aggregates data from multiple sources              | Fetch Demographic, Behavioral, Transactional Data | MCDM Scoring Engine                         | ## Parse Multi-Format Inputs: Standardizes data for AI and routing engines.                                    |
| MCDM Scoring Engine (AHP-TOPSIS) | Code                           | Calculates lead scores using AHP-TOPSIS MCDM      | Merge Lead Data Sources              | AI Lead Qualification Agent                | ## MCDN Route: Routes leads using multi-criteria decision making to assign priorities.                         |
| AI Lead Qualification Agent    | LangChain Agent                | Generates AI-driven lead qualification insights   | MCDM Scoring Engine, OpenRouter Chat Model, Lead Enrichment Tool | Prepare Lead Scores                      | ## AI Scores Lead Quality: Analyzes data to rank leads by conversion probability.                              |
| Lead Enrichment Tool           | LangChain toolCode             | Enriches lead data with tier, insights, and actions | AI Lead Qualification Agent (ai_tool) | AI Lead Qualification Agent (main)        | Same as above                                                                                                  |
| OpenRouter Chat Model          | LangChain LM                  | Provides Mistral-Saba AI language model support   | None                               | AI Lead Qualification Agent (ai_languageModel) | Same as above                                                                                                  |
| Prepare Lead Scores            | Set                             | Prepares unified scoring and routing fields       | AI Lead Qualification Agent          | Route by Lead Quality                      |                                                                                                               |
| Route by Lead Quality          | Switch                         | Routes leads to teams based on score thresholds   | Prepare Lead Scores                  | Assign to Enterprise, Mid-Market, SMB Teams, Send to Nurture Campaign | ## Assign to Specialized Team: Routes leads to correct sales teams by scoring and routing rules.               |
| Assign to Enterprise Sales Team | HTTP Request                   | Sends lead data to Enterprise sales team API      | Route by Lead Quality                | Collect Routing Results                    | Same as above                                                                                                  |
| Assign to Mid-Market Team      | HTTP Request                   | Sends lead data to Mid-Market sales team API      | Route by Lead Quality                | Collect Routing Results                    | Same as above                                                                                                  |
| Assign to SMB Team             | HTTP Request                   | Sends lead data to SMB sales team API              | Route by Lead Quality                | Collect Routing Results                    | Same as above                                                                                                  |
| Send to Nurture Campaign       | HTTP Request                   | Sends lead data to nurture campaign API            | Route by Lead Quality                | Collect Routing Results                    | Same as above                                                                                                  |
| Collect Routing Results        | Aggregate                      | Aggregates routing responses for CRM update        | Assign to Enterprise, Mid-Market, SMB Teams, Send to Nurture Campaign | Update CRM with Lead Scores              | ## Sync Assignment Metadata: Captures routing reasons and AI confidence for CRM and analytics.                |
| Update CRM with Lead Scores    | HTTP Request                   | Updates CRM with lead scores and routing metadata  | Collect Routing Results             | Calculate Performance KPIs                 | Same as above                                                                                                  |
| Calculate Performance KPIs     | Code                           | Computes KPIs on lead distribution, accuracy, etc. | Update CRM with Lead Scores         | Log KPIs to Analytics Dashboard            | ## Generate Performance Dashboard & Log: Compiles metrics for strategic visibility.                           |
| Log KPIs to Analytics Dashboard | HTTP Request                  | Sends computed KPIs to analytics dashboard API     | Calculate Performance KPIs          | None                                      | Same as above                                                                                                  |
| Sticky Note                   | Sticky Note                    | Documentation and guidance                          | None                              | None                                      | Multiple sticky notes provide documentation and context throughout the workflow.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval to trigger every hour (or preferred cadence).

2. **Create Set Node "Workflow Configuration"**  
   - Add string fields for:  
     - demographicApiUrl  
     - behavioralApiUrl  
     - transactionalApiUrl  
     - crmApiUrl  
     - analyticsApiUrl  
   - Add number fields for thresholds: enterpriseThreshold=85, midMarketThreshold=70, smbThreshold=50  
   - Add JSON object field for ahpWeights with keys demographic=0.3, behavioral=0.4, transactional=0.3 (adjust or expand if required)  
   - Connect Schedule Trigger output to this node input.

3. **Create HTTP Request Nodes for Data Fetching:**  
   - *Fetch Demographic Data*  
     - Method: GET (default)  
     - URL: Expression `{{$node["Workflow Configuration"].json.demographicApiUrl}}`  
     - Headers: Content-Type: application/json  
   - *Fetch Behavioral Data*  
     - Same config with behavioralApiUrl  
   - *Fetch Transactional Data*  
     - Same config with transactionalApiUrl  
   - Connect Workflow Configuration output to each of these nodes.

4. **Create Aggregate Node "Merge Lead Data Sources"**  
   - Mode: Aggregate All Item Data  
   - Destination Field Name: leadData  
   - Connect outputs of all three HTTP Request nodes into this node.

5. **Create Code Node "MCDM Scoring Engine (AHP-TOPSIS)"**  
   - Copy JavaScript code implementing the AHP-TOPSIS algorithm from the workflow  
   - Use criteria: companySize, budget, engagement, fitScore, urgency  
   - Load criteria weights from Workflow Configuration or use defaults  
   - Normalize, weight, calculate closeness coefficient, rank leads  
   - Connect Merge Lead Data Sources output to this node.

6. **Set Up LangChain Integration:**  
   - Create LangChain LM node "OpenRouter Chat Model"  
     - Model: mistralai/mistral-saba  
     - Provide OpenRouter API credentials  
   - Create LangChain toolCode node "Lead Enrichment Tool"  
     - JavaScript code to parse lead JSON, compute quality tiers, and add insights  
   - Create LangChain Agent node "AI Lead Qualification Agent"  
     - Define prompt as expert lead qualification analyst  
     - Use OpenRouter Chat Model as AI language model  
     - Use Lead Enrichment Tool for enrichment  
   - Connect MCDM Scoring Engine output to AI Lead Qualification Agent input.

7. **Create Set Node "Prepare Lead Scores"**  
   - Map fields from AI output: leadScore, leadTier, aiInsights, routingDecision  
   - Connect AI Lead Qualification Agent to this node.

8. **Create Switch Node "Route by Lead Quality"**  
   - Define routing rules comparing leadScore against thresholds from Workflow Configuration  
   - Output branches: Enterprise, Mid-Market, SMB, Nurture

9. **Create HTTP Request Nodes for Lead Assignment:**  
   - Assign to Enterprise Sales Team  
     - Method: POST  
     - URL: placeholder `<__PLACEHOLDER_VALUE__Enterprise Sales Team Assignment API__>`  
     - Body includes leadId, leadData, score, tier, aiInsights  
   - Assign to Mid-Market Team (similar with Mid-Market API)  
   - Assign to SMB Team (similar with SMB API)  
   - Send to Nurture Campaign  
     - Method: POST  
     - URL: placeholder `<__PLACEHOLDER_VALUE__Nurture Campaign API__>`  
     - JSON body with leadId, email, name, score, qualificationStatus, nurtureRecommendations, assignedCampaign, timestamp  
   - Connect Switch node outputs to corresponding assignment nodes.

10. **Create Aggregate Node "Collect Routing Results"**  
    - Aggregate all routing assignment outputs  
    - Connect all assignment nodes to this node.

11. **Create HTTP Request Node "Update CRM with Lead Scores"**  
    - Method: POST  
    - URL from Workflow Configuration `crmApiUrl`  
    - JSON body includes leadId, leadScore, tier, routingDecision, assignedTeam, aiInsights, and timestamp  
    - Connect Collect Routing Results output to this node.

12. **Create Code Node "Calculate Performance KPIs"**  
    - Implement JavaScript code to:  
      - Count leads per tier  
      - Calculate average scores, conversion rates, model accuracy  
      - Compute lead distribution and performance metrics  
    - Connect Update CRM with Lead Scores output to this node.

13. **Create HTTP Request Node "Log KPIs to Analytics Dashboard"**  
    - Method: POST  
    - URL from Workflow Configuration `analyticsApiUrl`  
    - JSON body containing KPIs and timestamp  
    - Connect Calculate Performance KPIs output to this node.

14. **Add Sticky Notes as Documentation Nodes**  
    - Provide explanations for each block, setup instructions, use cases, and benefits.

15. **Configure Credentials**  
    - OpenRouter API credentials for LangChain LM node  
    - API keys or OAuth tokens for all HTTP Request nodes (CRM, team assignment APIs, analytics)  
    - Ensure all endpoints are reachable and authenticated.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                    |
|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Workflow employs Mistral-Saba AI model via OpenRouter for lead qualification insights                                   | Requires OpenRouter API credentials                |
| Uses AHP-TOPSIS MCDM algorithm for transparent multi-criteria lead scoring                                             | Code node includes detailed implementation        |
| Routing thresholds and AHP weights configurable in Workflow Configuration node                                         | Allows tuning by sales strategy                     |
| Supports CRM integration with flexible API endpoint configuration                                                     | Placeholder URLs need replacement with real APIs  |
| Provides KPI logging for analytics dashboard integration                                                                | Enables ongoing performance monitoring             |
| Sticky notes throughout workflow document setup, logic, use cases, and customization tips                              | Enhances maintainability and user understanding    |
| Recommended to secure API endpoints and use retry logic in HTTP nodes for production robustness                        | To handle transient network or API errors          |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.