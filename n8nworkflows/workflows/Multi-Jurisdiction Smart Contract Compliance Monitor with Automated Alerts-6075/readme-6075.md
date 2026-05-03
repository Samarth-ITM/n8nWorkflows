Multi-Jurisdiction Smart Contract Compliance Monitor with Automated Alerts

https://n8nworkflows.xyz/workflows/multi-jurisdiction-smart-contract-compliance-monitor-with-automated-alerts-6075


# Multi-Jurisdiction Smart Contract Compliance Monitor with Automated Alerts

### 1. Workflow Overview

This workflow automates the monitoring of legal compliance risks for active contracts across multiple jurisdictions (EU, US, UK, France). It periodically fetches regulatory updates from official sources, analyzes their impact on existing contracts, assesses risk levels, and sends automated alerts to legal and compliance teams accordingly.

Logical blocks in this workflow:

- **1.1 Scheduled Trigger & Configuration:** The workflow initiates on weekdays at 8 AM, loads compliance thresholds, jurisdictions, and notification settings.
  
- **1.2 Regulatory Data Collection:** Concurrent HTTP requests gather recent legal and regulatory updates from multiple jurisdictional sources (EU, US, UK).

- **1.3 Active Contracts Retrieval:** Queries the internal database to fetch all active contracts with details relevant to compliance.

- **1.4 Compliance Impact Analysis:** A custom JavaScript node processes regulatory updates and contracts to identify affected contracts, calculate risk scores, assign risk levels, and recommend actions.

- **1.5 Risk-based Filtering & Alerts:** The analyzed results are filtered by risk levels (critical, high, medium). For each level, tailored email alerts are sent to appropriate legal and compliance contacts.

- **1.6 Logging:** All compliance check results are logged into a Google Sheet for record-keeping and auditing.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Scheduled Trigger & Configuration

- **Overview:**  
  Initiates the workflow on weekdays at 8:00 AM and sets up core compliance parameters such as risk thresholds, jurisdictions, and notification emails.

- **Nodes Involved:**  
  - Daily Compliance Check (Schedule Trigger)  
  - Compliance Settings (Set)  
  - Sticky Note (Documentation)

- **Node Details:**

  - **Daily Compliance Check**  
    - Type: Schedule Trigger  
    - Configuration: Runs at 08:00 on Monday to Friday using a cron expression `0 8 * * 1-5`.  
    - Input: None (trigger node)  
    - Output: Triggers the workflow flow to Compliance Settings node.  
    - Edge Cases: Cron misconfiguration could prevent triggering; ensure server timezone matches expectations.

  - **Compliance Settings**  
    - Type: Set  
    - Configuration: Defines numeric risk thresholds (medium=31, high=61, critical=81), jurisdictions monitored ("EU,US,UK,FR"), and emails for legal and compliance officers.  
    - Key Expressions: Static values; jurisdictions split later in code.  
    - Input: Trigger from Daily Compliance Check  
    - Output: Initiates regulatory monitoring HTTP requests.  
    - Edge Cases: Missing or malformed email addresses or jurisdiction strings may cause errors downstream.

  - **Sticky Note**  
    - Type: Sticky Note (documentation)  
    - Content: Explains configuration parameters for compliance monitoring setup.  
    - No functional input/output.

---

#### Block 1.2: Regulatory Data Collection

- **Overview:**  
  Gathers regulatory updates from three authoritative sources for EU, US, and UK jurisdictions via HTTP requests configured with query parameters based on current date.

- **Nodes Involved:**  
  - Monitor EU Regulations (HTTP Request)  
  - Monitor US Federal Register (HTTP Request)  
  - Monitor UK Legislation (HTTP Request)

- **Node Details:**

  - **Monitor EU Regulations**  
    - Type: HTTP Request  
    - URL: `https://eur-lex.europa.eu/legal-content/EN/ALL/`  
    - Method: GET  
    - Query Parameters: Includes current year and month dynamically, `qid=regulatory_updates`, `SUBDOM_INIT=LEGISLATION`.  
    - Input: From Compliance Settings  
    - Output: JSON response with EU legislative updates passed to analysis node.  
    - Edge Cases: Possible HTTP timeout, rate limiting, or unexpected response structure.

  - **Monitor US Federal Register**  
    - Type: HTTP Request  
    - URL: `https://www.federalregister.gov/api/v1/documents.json`  
    - Method: GET  
    - Query Parameters: Fetches newest 20 documents published within last 7 days; fields include title, URL, date, abstract.  
    - Input: From Compliance Settings  
    - Output: JSON with recent US federal regulatory documents.  
    - Edge Cases: API limits, date query formatting issues, empty results.

  - **Monitor UK Legislation**  
    - Type: HTTP Request  
    - URL: `https://www.legislation.gov.uk/new`  
    - Method: GET  
    - Query Parameters: Limit 20, JSON format.  
    - Input: From Compliance Settings  
    - Output: JSON with UK legislation updates.  
    - Edge Cases: Possible changes in API response format or URL availability.

---

#### Block 1.3: Active Contracts Retrieval

- **Overview:**  
  Queries the internal PostgreSQL database to retrieve all active contracts with relevant metadata for compliance evaluation.

- **Nodes Involved:**  
  - Get Active Contracts (Postgres)

- **Node Details:**

  - **Get Active Contracts**  
    - Type: Postgres  
    - Query: Selects contract_id, type, jurisdiction, signing and expiry dates, key clauses, compliance tags from 'contracts' table where status = 'active'.  
    - Input: From Compliance Settings node (parallel with regulatory monitors)  
    - Output: JSON array of contracts passed to analysis node.  
    - Edge Cases: Database connection failure, query errors, empty result set.

---

#### Block 1.4: Compliance Impact Analysis

- **Overview:**  
  Custom JavaScript logic to merge regulatory updates and active contracts, identify affected contracts by matching jurisdictions and impact keywords, compute risk scores based on multiple factors, and assign risk levels with recommended actions.

- **Nodes Involved:**  
  - Analyze Compliance Impact (Code)

- **Node Details:**

  - **Analyze Compliance Impact**  
    - Type: Code (JavaScript)  
    - Inputs: JSON from all regulatory monitors and active contracts node.  
    - Main Tasks:  
      - Extract impact areas from texts using keyword matching.  
      - Match contracts to regulatory updates by jurisdiction and clause/tag relevance.  
      - Calculate risk score considering contract count, impact area priority, recency of update, contract expiry urgency.  
      - Assign risk level: low, medium, high, critical based on thresholds from Compliance Settings.  
      - Prepare detailed impact assessment and recommended actions.  
    - Key Variables: `complianceSettings`, `regulatoryUpdates`, `contracts`, `complianceAnalysis`.  
    - Output: Array of compliance risk analyses for further filtering.  
    - Edge Cases: Missing data keys, inconsistent data formats, empty arrays, date parsing errors.  
    - Notes: Implements a helper function to extract impact areas with domain-specific keywords.

---

#### Block 1.5: Risk-based Filtering & Alerts

- **Overview:**  
  Filters the analyzed compliance results by risk level and sends customized email alerts with tailored HTML content to stakeholders.

- **Nodes Involved:**  
  - Filter Critical Compliance Issues (If)  
  - Send Critical Legal Alert (Gmail)  
  - Filter High Risk Issues (If)  
  - Send High Risk Alert (Gmail)  
  - Filter Medium Risk Issues (If)  
  - Send Medium Risk Alert (Gmail)  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Filter Critical Compliance Issues**  
    - Type: If  
    - Condition: `$json.risk_level == "critical"`  
    - Input: From Analyze Compliance Impact  
    - Output: Passes critical risks to Send Critical Legal Alert  
    - Edge Cases: Case sensitivity, missing risk_level property.

  - **Send Critical Legal Alert**  
    - Type: Gmail  
    - Configuration:  
      - To: legal email from Compliance Settings  
      - Subject: "🚨 CRITICAL COMPLIANCE ALERT - Immediate Legal Review Required"  
      - Body: Rich HTML with styled sections detailing regulatory update, affected contracts, impact assessment, and immediate action steps, including links and contact info.  
    - Input: Filter Critical Compliance Issues node  
    - Edge Cases: SMTP authentication failures, invalid recipient address.

  - **Filter High Risk Issues**  
    - Type: If  
    - Condition: `$json.risk_level == "high"`  
    - Output: Passes to Send High Risk Alert

  - **Send High Risk Alert**  
    - Type: Gmail  
    - Configuration:  
      - To: legal email from Compliance Settings  
      - Subject: "🟠 High Priority Compliance Review - [Jurisdiction]"  
      - Body: HTML email emphasizing legal review within 72 hours with regulatory summary, contract impact, and recommended timeline.  
    - Edge Cases: Same as above.

  - **Filter Medium Risk Issues**  
    - Type: If  
    - Condition: `$json.risk_level == "medium"`  
    - Output: Passes to Send Medium Risk Alert

  - **Send Medium Risk Alert**  
    - Type: Gmail  
    - Configuration:  
      - To: compliance officer email from Compliance Settings  
      - Subject: "🟡 Compliance Monitoring Update - [Jurisdiction]"  
      - Body: HTML email recommending scheduled review within two weeks, with update summary and monitoring plan.  
    - Edge Cases: Same as above.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Summarizes automated responses by risk level for legal compliance team  
    - No functional input/output.

---

#### Block 1.6: Logging

- **Overview:**  
  Logs each compliance check and its risk assessment details into a Google Sheet for historical tracking and auditing.

- **Nodes Involved:**  
  - Log Compliance Check (Google Sheets)

- **Node Details:**

  - **Log Compliance Check**  
    - Type: Google Sheets  
    - Operation: Append Row  
    - Document ID: Placeholder `"your-google-sheet-id"` (must be replaced)  
    - Sheet Name: "Compliance Monitoring Log"  
    - Values: Regulatory update title, jurisdiction, risk score, risk level, contract count, recommended action, timestamp  
    - Input: From Analyze Compliance Impact (all risk levels)  
    - Edge Cases: Invalid document ID, permission errors, network issues.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                         | Input Node(s)                | Output Node(s)                              | Sticky Note                                                                                                                                                    |
|-----------------------------|--------------------|---------------------------------------|-----------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Compliance Check       | Schedule Trigger   | Initiates workflow on weekdays 8 AM   | None                        | Compliance Settings                         |                                                                                                                                                               |
| Compliance Settings          | Set                | Sets risk thresholds, jurisdictions, and emails | Daily Compliance Check      | Monitor EU Regulations, Monitor US Federal Register, Monitor UK Legislation, Get Active Contracts |                                                                                                                                                               |
| Sticky Note                 | Sticky Note         | Documentation of compliance setup      | None                        | None                                        | ## Compliance Monitor Setup ⚙️ **Configure monitoring:** - Jurisdictions to monitor - Contract types and categories - Risk thresholds - Legal team notifications |
| Monitor EU Regulations       | HTTP Request       | Fetches EU legislative updates         | Compliance Settings         | Analyze Compliance Impact                   |                                                                                                                                                               |
| Monitor US Federal Register  | HTTP Request       | Fetches US federal regulatory updates  | Compliance Settings         | Analyze Compliance Impact                   |                                                                                                                                                               |
| Monitor UK Legislation       | HTTP Request       | Fetches UK legislative updates          | Compliance Settings         | Analyze Compliance Impact                   |                                                                                                                                                               |
| Get Active Contracts         | Postgres           | Queries active contracts from DB       | Compliance Settings         | Analyze Compliance Impact                   |                                                                                                                                                               |
| Analyze Compliance Impact    | Code (JavaScript)  | Analyzes updates vs contracts, calculates risk | Monitor EU Regulations, Monitor US Federal Register, Monitor UK Legislation, Get Active Contracts | Filter Critical Compliance Issues, Filter High Risk Issues, Filter Medium Risk Issues, Log Compliance Check |                                                                                                                                                               |
| Filter Critical Compliance Issues | If             | Filters critical risk analyses          | Analyze Compliance Impact   | Send Critical Legal Alert                    |                                                                                                                                                               |
| Send Critical Legal Alert    | Gmail              | Sends emergency alert email for critical risks | Filter Critical Compliance Issues | None                                        |                                                                                                                                                               |
| Filter High Risk Issues      | If                 | Filters high risk analyses              | Analyze Compliance Impact   | Send High Risk Alert                         |                                                                                                                                                               |
| Send High Risk Alert         | Gmail              | Sends high priority alert email         | Filter High Risk Issues     | None                                        |                                                                                                                                                               |
| Filter Medium Risk Issues    | If                 | Filters medium risk analyses            | Analyze Compliance Impact   | Send Medium Risk Alert                       |                                                                                                                                                               |
| Send Medium Risk Alert       | Gmail              | Sends scheduled review alert email      | Filter Medium Risk Issues   | None                                        |                                                                                                                                                               |
| Sticky Note1                | Sticky Note         | Documentation of automated legal responses | None                        | None                                        | ## Legal Compliance ⚖️ **Automated responses:** - Critical: Emergency legal review - High: Priority compliance check - Medium: Scheduled review - Low: Routine monitoring |
| Log Compliance Check         | Google Sheets      | Logs compliance check results           | Analyze Compliance Impact   | None                                        |                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the trigger node:**  
   - Add a **Schedule Trigger** node named "Daily Compliance Check".  
   - Set its cron expression to `0 8 * * 1-5` to trigger at 8:00 AM Monday to Friday.

2. **Set compliance parameters:**  
   - Add a **Set** node named "Compliance Settings".  
   - Configure numeric fields:  
     - `mediumRiskThreshold` = 31  
     - `highRiskThreshold` = 61  
     - `criticalRiskThreshold` = 81  
   - Configure string fields:  
     - `jurisdictions` = "EU,US,UK,FR"  
     - `legalEmail` = "legal@company.com" (replace with actual)  
     - `complianceOfficer` = "compliance@company.com" (replace with actual)  
   - Connect "Daily Compliance Check" → "Compliance Settings".

3. **Add documentation sticky note:**  
   - Add a **Sticky Note** node near the "Compliance Settings" node with the content describing compliance monitor setup.

4. **Add HTTP Request nodes for regulatory data:**  
   - Create three **HTTP Request** nodes named:  
     - "Monitor EU Regulations"  
     - "Monitor US Federal Register"  
     - "Monitor UK Legislation"  
   - Configure each as follows:

     - **Monitor EU Regulations:**  
       - Method: GET  
       - URL: `https://eur-lex.europa.eu/legal-content/EN/ALL/`  
       - Query Params:  
         - `qid` = "regulatory_updates"  
         - `DD_YEAR` = `{{ new Date().getFullYear() }}`  
         - `DD_MONTH` = `{{ String(new Date().getMonth() + 1).padStart(2, '0') }}`  
         - `SUBDOM_INIT` = "LEGISLATION"

     - **Monitor US Federal Register:**  
       - Method: GET  
       - URL: `https://www.federalregister.gov/api/v1/documents.json`  
       - Query Params:  
         - `order` = "newest"  
         - `fields[]` = "title,html_url,publication_date,abstract"  
         - `per_page` = 20  
         - `conditions[publication_date][gte]` = `{{ new Date(Date.now() - 7*24*60*60*1000).toISOString().split('T')[0] }}`

     - **Monitor UK Legislation:**  
       - Method: GET  
       - URL: `https://www.legislation.gov.uk/new`  
       - Query Params:  
         - `limit` = 20  
         - `format` = "json"

   - Connect "Compliance Settings" to all three HTTP Request nodes in parallel.

5. **Add database query node:**  
   - Add a **Postgres** node named "Get Active Contracts".  
   - Configure credentials for your Postgres database.  
   - Query:  
     ```sql
     SELECT contract_id, contract_type, jurisdiction, signing_date, expiry_date, key_clauses, compliance_tags FROM contracts WHERE status = 'active'
     ```  
   - Connect "Compliance Settings" → "Get Active Contracts".

6. **Add Code node for compliance analysis:**  
   - Add a **Code** node named "Analyze Compliance Impact".  
   - In the node, implement the JavaScript logic to:  
     - Parse inputs from all regulatory nodes and contracts.  
     - Extract impact areas by keyword matching.  
     - Match contracts by jurisdiction and clause/tag relevance.  
     - Compute risk scores using thresholds and weighted factors.  
     - Assign risk levels and recommended actions.  
     - Output an array of analysis results filtered for risk_score > 0.  
   - Connect all 4 data nodes ("Monitor EU Regulations", "Monitor US Federal Register", "Monitor UK Legislation", "Get Active Contracts") to this node.

7. **Add filtering nodes for risk levels:**  
   - Add three **If** nodes named:  
     - "Filter Critical Compliance Issues" with condition: `$json.risk_level == "critical"`  
     - "Filter High Risk Issues" with condition: `$json.risk_level == "high"`  
     - "Filter Medium Risk Issues" with condition: `$json.risk_level == "medium"`  
   - Connect "Analyze Compliance Impact" → each filter node.

8. **Add Gmail nodes for alerts:**  
   - Add three **Gmail** nodes named:  
     - "Send Critical Legal Alert"  
     - "Send High Risk Alert"  
     - "Send Medium Risk Alert"

   - Configure each Gmail node:  
     - Set recipient email using expressions from Compliance Settings node.  
     - Compose HTML email bodies as per the provided styling and content.  
     - Use dynamic subject lines reflecting risk and jurisdiction.  
   - Connect each filter node to its corresponding Gmail node.

9. **Add a sticky note describing automated legal responses:**  
   - Insert a **Sticky Note** near alert nodes with summarized automated responses by risk level.

10. **Add Google Sheets node for logging:**  
    - Add a **Google Sheets** node named "Log Compliance Check".  
    - Configure OAuth2 credentials for Google Sheets.  
    - Document ID: set your Google Sheet's ID.  
    - Sheet Name: "Compliance Monitoring Log"  
    - Operation: Append Row  
    - Values: Map relevant fields such as regulatory update title, jurisdiction, risk score, risk level, contract count, recommended action, and timestamp.  
    - Connect "Analyze Compliance Impact" → "Log Compliance Check".

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Compliance Monitor Setup: Jurisdictions, contract types, risk thresholds, legal notifications must be configured properly | See Sticky Note near "Compliance Settings" node                                                  |
| Legal Compliance Automated Responses: Critical = emergency review; High = priority check; Medium = scheduled review    | See Sticky Note near alert Gmail nodes                                                          |
| Google Sheets Document ID must be replaced with the actual sheet ID for logging                                        | Ensure Google Sheets credentials have write access                                              |
| Gmail nodes require OAuth2 credentials with sending permissions                                                        | Configure Gmail credentials in n8n for alert delivery                                           |
| API endpoints for regulatory data may change; monitor for API deprecation or changes                                   | EU Lex: https://eur-lex.europa.eu/legal-content/EN/ALL/; US Fed Register API; UK Legislation API |
| Risk thresholds can be adjusted in the "Compliance Settings" node                                                      | Adjust numeric values depending on organizational risk tolerance                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.