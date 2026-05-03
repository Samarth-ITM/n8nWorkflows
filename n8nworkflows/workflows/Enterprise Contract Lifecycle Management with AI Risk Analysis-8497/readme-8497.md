Enterprise Contract Lifecycle Management with AI Risk Analysis

https://n8nworkflows.xyz/workflows/enterprise-contract-lifecycle-management-with-ai-risk-analysis-8497


# Enterprise Contract Lifecycle Management with AI Risk Analysis

### 1. Workflow Overview

This workflow automates the Enterprise Contract Lifecycle Management (CLM) process by integrating multiple contract intake channels, AI-powered contract data extraction, risk analysis, and compliance scoring. It also manages approval routing, updates CRM systems, tracks obligations, and issues alerts for contract deadlines. The workflow is divided into the following logical blocks:

- **1.1 Multi-Channel Contract Intake:** Listens for new contracts arriving via email, Google Drive, and Salesforce CRM.
- **1.2 Contract Deduplication & Validation:** Merges intake sources and prevents duplicate contract processing.
- **1.3 AI-Powered Contract Data Extraction:** Uses advanced AI (PDF Vector nodes) to extract detailed contract metadata and perform risk analysis.
- **1.4 Contract Template Retrieval:** Fetches standard clauses/templates for comparison.
- **1.5 Comprehensive Contract Analysis & Scoring:** Aggregates AI data, template info, and CRM data to score risk and determine approval levels.
- **1.6 Contract Repository Update & Approval Routing:** Saves contract data, routes high-risk contracts to legal, and updates Salesforce CRM.
- **1.7 Obligation Tracking & Alerts:** Daily scheduled checks for upcoming contract deadlines and sends Slack summaries.
- **1.8 Analytics & Dashboard Note:** Provides key contract statistics and links to a dashboard.

---

### 2. Block-by-Block Analysis

#### 2.1 Multi-Channel Contract Intake

**Overview:**  
This block captures new contracts from multiple channels: monitored email inbox, Google Drive folder, and Salesforce CRM updates. It triggers the downstream processing upon contract arrival.

**Nodes Involved:**  
- Monitor Contract Emails (email-read IMAP)  
- Monitor Google Drive (Google Drive Trigger)  
- Hourly CRM Check (Schedule Trigger)  
- Check Salesforce (Salesforce node)  
- Merge Contract Sources (Merge node)

**Node Details:**

- **Monitor Contract Emails**  
  - Type: Email Read IMAP  
  - Role: Polls the specified mailbox every minute for incoming contract emails.  
  - Config: Uses custom email config for contractreview@company.com, monitoring INBOX folder.  
  - Inputs: None (trigger node)  
  - Outputs: Contract email data forwarded to merge node.  
  - Edge Cases: Email server authentication failure, connectivity issues, large attachments.  

- **Monitor Google Drive**  
  - Type: Google Drive Trigger  
  - Role: Watches designated folder "contracts-intake" for new or updated files.  
  - Inputs: None (trigger node)  
  - Outputs: New/updated contract files forwarded to merge node.  
  - Edge Cases: Google API quota limits, folder ID changes, permission errors.  

- **Hourly CRM Check**  
  - Type: Schedule Trigger  
  - Role: Runs hourly to check Salesforce for contracts in "Contract Review" stage modified within last hour.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers Salesforce query node.  
  - Edge Cases: Scheduler downtime, CRM API limits.  

- **Check Salesforce**  
  - Type: Salesforce node  
  - Role: Retrieves contract opportunities in "Contract Review" stage updated in last hour.  
  - Config: Filters on stage and lastModified time.  
  - Inputs: Trigger from schedule node.  
  - Outputs: Contract CRM data forwarded to merge node.  
  - Edge Cases: Salesforce API limits, authentication failures, data inconsistencies.  

- **Merge Contract Sources**  
  - Type: Merge (multiplex mode)  
  - Role: Combines contract data from email, Drive, and CRM into unified stream.  
  - Inputs: From Monitor Email, Google Drive, and Salesforce check nodes.  
  - Outputs: Combined contract data forwarded for duplicate checking.  
  - Edge Cases: Data format mismatches, empty inputs from any channel.

---

#### 2.2 Contract Deduplication & Validation

**Overview:**  
Ensures contracts are not reprocessed by checking contract hash against the database. Routes only new contracts for further AI processing.

**Nodes Involved:**  
- Check Duplicate (Postgres)  
- New Contract? (If node)

**Node Details:**

- **Check Duplicate**  
  - Type: Postgres  
  - Role: Queries contract repository for existing contract IDs matching the file hash.  
  - Config: SQL query uses fileHash from incoming data for lookup.  
  - Inputs: From Merge Contract Sources.  
  - Outputs: Contract data with possible existing contract_id.  
  - Edge Cases: DB connection issues, missing fileHash, SQL injection risks.  

- **New Contract?**  
  - Type: If  
  - Role: Checks if contract_id is empty (new contract) or existing.  
  - Inputs: From Check Duplicate.  
  - Outputs: True branch proceeds to AI extraction; false branch discards or stops.  
  - Edge Cases: Unexpected data types, empty inputs.

---

#### 2.3 AI-Powered Contract Data Extraction

**Overview:**  
Uses PDF Vector AI nodes to extract detailed contract information and perform multi-dimensional risk analysis.

**Nodes Involved:**  
- PDF Vector - Extract All Data  
- PDF Vector - Risk Analysis

**Node Details:**

- **PDF Vector - Extract All Data**  
  - Type: PDF Vector Extract  
  - Role: Extracts 50+ detailed contract fields including parties, terms, obligations, clauses, and metadata.  
  - Config: Uses a comprehensive JSON schema and detailed prompt for contract data extraction from the contract file binary.  
  - Inputs: From New Contract? node (true branch).  
  - Outputs: JSON contract data forwarded to risk analysis and template retrieval.  
  - Edge Cases: Poor document quality, OCR errors, unexpected formats, AI model failures.  

- **PDF Vector - Risk Analysis**  
  - Type: PDF Vector Ask  
  - Role: Performs comprehensive risk scoring across legal, financial, operational, compliance, and relationship dimensions.  
  - Config: Prompt specifying risk categories and scoring instructions; returns risk scores 1-10 and overall assessment.  
  - Inputs: From PDF Vector Extract All Data node (binary file input).  
  - Outputs: Risk analysis JSON forwarded to analysis engine.  
  - Edge Cases: AI latency, partial or unclear risk outputs, prompt misinterpretation.

---

#### 2.4 Contract Template Retrieval

**Overview:**  
Queries the database to retrieve the active standard contract template matching the extracted contract type for compliance comparison.

**Nodes Involved:**  
- Get Contract Template (Postgres)

**Node Details:**

- **Get Contract Template**  
  - Type: Postgres  
  - Role: Fetches active contract template and clauses based on contractType extracted by AI.  
  - Inputs: From PDF Vector Extract All Data node.  
  - Outputs: Template JSON forwarded to analysis engine.  
  - Edge Cases: Missing template for new contract type, DB query errors.

---

#### 2.5 Comprehensive Contract Analysis & Scoring

**Overview:**  
Combines extracted contract data, AI risk analysis, template info, and CRM opportunity data to calculate a detailed risk score, identify missing clauses, determine approval routing, and generate alerts.

**Nodes Involved:**  
- Analyze & Score Contract (Code node)

**Node Details:**

- **Analyze & Score Contract**  
  - Type: Code (JavaScript)  
  - Role: Implements business logic for scoring contracts by evaluating deal size, missing clauses, payment terms, auto-renewal risks, and integrates AI risk scores. Determines approval level (Manager, Director, VP, Legal) and creates alerts for expirations and deliverables.  
  - Inputs:  
    - Extracted contract data (from PDF Vector Extract)  
    - AI risk analysis (from PDF Vector Risk Analysis)  
    - Contract template (from Postgres template query)  
    - CRM opportunity data (from Salesforce)  
  - Outputs: Comprehensive JSON object with contract analysis, risk assessment, approval routing, alerts, CRM alignment, metadata, and full extraction data.  
  - Edge Cases: Missing or malformed inputs, date parsing errors, division by zero in scoring, logic misconfigurations.

---

#### 2.6 Contract Repository Update & Approval Routing

**Overview:**  
Saves the analyzed contract data in the central Postgres contract repository, evaluates if legal review is required, sends Slack notifications for high-risk contracts, and updates Salesforce CRM opportunity records.

**Nodes Involved:**  
- Save to Contract Repository (Postgres)  
- Needs Legal Review? (If)  
- Notify Legal Team (Slack)  
- Update Salesforce (Salesforce node)

**Node Details:**

- **Save to Contract Repository**  
  - Type: Postgres  
  - Role: Inserts new contract record with detailed fields including extracted data, risk scores, approval level, alerts, CRM data, timestamps, and status "pending_approval".  
  - Inputs: From Analyze & Score Contract node.  
  - Outputs: Connects to Needs Legal Review? and Update Salesforce nodes.  
  - Edge Cases: DB connection failure, constraint violations, JSON serialization errors.  

- **Needs Legal Review?**  
  - Type: If  
  - Role: Checks if risk level is "High" or if there are missing critical clauses to determine if legal review is needed.  
  - Inputs: From Save to Contract Repository.  
  - Outputs: True branch to Notify Legal Team, false branch ends flow.  
  - Edge Cases: Incorrect riskLevel or missingClauses data types.  

- **Notify Legal Team**  
  - Type: Slack  
  - Role: Sends a formatted Slack message to #legal-contracts channel with contract type, parties, value, risk score, risk factors, missing clauses, and a link to contract review portal.  
  - Inputs: From Needs Legal Review? (true branch).  
  - Outputs: None (end node).  
  - Edge Cases: Slack API rate limits, channel permissions, formatting failures.  

- **Update Salesforce**  
  - Type: Salesforce node  
  - Role: Updates opportunity record fields with contract ID, status ("Under Review"), expiration date, and risk score to keep CRM aligned.  
  - Inputs: From Save to Contract Repository.  
  - Outputs: None (end node).  
  - Edge Cases: API auth failures, permission issues, record lock conflicts.

---

#### 2.7 Obligation Tracking & Alerts

**Overview:**  
Periodically checks contracts with upcoming deadlines and sends daily summaries of alerts to a Slack channel for contract management teams.

**Nodes Involved:**  
- Daily Alert Check (Schedule Trigger)  
- Get Upcoming Alerts (Postgres)  
- Send Daily Summary (Slack)

**Node Details:**

- **Daily Alert Check**  
  - Type: Schedule Trigger  
  - Role: Triggers daily to initiate alert checks.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers Get Upcoming Alerts.  
  - Edge Cases: Scheduler downtime.  

- **Get Upcoming Alerts**  
  - Type: Postgres  
  - Role: Queries active contracts with alerts due between current date and next 30 days, expands alerts JSON array.  
  - Inputs: From Daily Alert Check.  
  - Outputs: Contracts with alert details forwarded to Slack node.  
  - Edge Cases: DB performance issues, alert data inconsistencies.  

- **Send Daily Summary**  
  - Type: Slack  
  - Role: Posts a daily digest message in "contract-alerts" Slack channel listing upcoming contract deadlines with type, description, due date, and days remaining.  
  - Inputs: From Get Upcoming Alerts.  
  - Outputs: None (end node).  
  - Edge Cases: Slack API limits, message formatting errors.

---

#### 2.8 Analytics & Dashboard Note

**Overview:**  
Provides a sticky note summarizing high-level contract analytics with a link to the company’s contract analytics dashboard.

**Nodes Involved:**  
- Analytics Dashboard (Sticky Note)

**Node Details:**

- **Analytics Dashboard**  
  - Type: Sticky Note  
  - Role: Displays total contracts, total value, count of high-risk contracts, expiring soon contracts, and dashboard URL.  
  - Inputs: None connected (static or dynamically updated elsewhere).  
  - Outputs: None.  
  - Edge Cases: Data refresh latency if connected dynamically.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                       | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                          |
|----------------------------|---------------------------|------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow Overview          | Sticky Note               | Describes workflow purpose         | None                          | None                          | ## Enterprise Contract Lifecycle Management… (overview content)                                    |
| Monitor Contract Emails    | Email Read IMAP           | Polls contract emails              | None                          | Merge Contract Sources         | Monitor email for contracts                                                                        |
| Monitor Google Drive       | Google Drive Trigger      | Watches contract folder            | None                          | Merge Contract Sources         | Watch for new contracts                                                                            |
| Hourly CRM Check           | Schedule Trigger          | Checks CRM hourly                  | None                          | Check Salesforce               | Check CRM for contracts                                                                            |
| Check Salesforce           | Salesforce                | Retrieves contracts from CRM       | Hourly CRM Check              | Merge Contract Sources         | Get contracts from CRM                                                                             |
| Merge Contract Sources     | Merge                    | Combines all contract inputs       | Monitor Contract Emails, Monitor Google Drive, Check Salesforce | Check Duplicate                |                                                                                                    |
| Check Duplicate            | Postgres                  | Prevents duplicate processing      | Merge Contract Sources        | New Contract?                  | Prevent reprocessing                                                                              |
| New Contract?              | If                        | Checks if contract is new          | Check Duplicate               | PDF Vector - Extract All Data  |                                                                                                    |
| PDF Vector - Extract All Data | PDF Vector Extract       | Extracts detailed contract data   | New Contract?                 | PDF Vector - Risk Analysis, Get Contract Template | AI extraction of 50+ fields                                                                       |
| PDF Vector - Risk Analysis | PDF Vector Ask            | Performs risk scoring              | PDF Vector - Extract All Data | Analyze & Score Contract       | Multi-dimensional risk scoring                                                                    |
| Get Contract Template      | Postgres                  | Retrieves contract template        | PDF Vector - Extract All Data | Analyze & Score Contract       | Retrieve standard template                                                                        |
| Analyze & Score Contract   | Code                      | Analyzes and scores contract       | PDF Vector - Extract All Data, PDF Vector - Risk Analysis, Get Contract Template, Check Salesforce | Save to Contract Repository    | Comprehensive analysis engine                                                                     |
| Save to Contract Repository| Postgres                  | Saves contract data                | Analyze & Score Contract      | Needs Legal Review?, Update Salesforce | Central contract database                                                                        |
| Needs Legal Review?        | If                        | Determines if legal review needed  | Save to Contract Repository   | Notify Legal Team (if true)    |                                                                                                    |
| Notify Legal Team          | Slack                     | Sends alert to legal team          | Needs Legal Review?           | None                          | High risk alert                                                                                   |
| Update Salesforce          | Salesforce                | Updates CRM opportunity            | Save to Contract Repository   | None                          | Sync with CRM                                                                                    |
| Daily Alert Check          | Schedule Trigger          | Triggers daily alert checks        | None                          | Get Upcoming Alerts            | Check for upcoming deadlines                                                                     |
| Get Upcoming Alerts        | Postgres                  | Retrieves contracts with alerts    | Daily Alert Check             | Send Daily Summary             | Find contracts with deadlines                                                                    |
| Send Daily Summary         | Slack                     | Sends daily alert summary          | Get Upcoming Alerts           | None                          | Daily deadline digest                                                                            |
| Analytics Dashboard        | Sticky Note               | Displays contract analytics        | None                          | None                          | =Contract Analytics Dashboard… (dashboard link included)                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note:**  
   - Name: Workflow Overview  
   - Content: High-level description of the workflow purpose and blocks.

2. **Add Email Trigger Node:**  
   - Name: Monitor Contract Emails  
   - Type: Email Read IMAP  
   - Configuration: Poll INBOX every minute with custom email config for contractreview@company.com.

3. **Add Google Drive Trigger Node:**  
   - Name: Monitor Google Drive  
   - Type: Google Drive Trigger  
   - Configuration: Watch folder with ID "contracts-intake" for created/updated files.

4. **Add Schedule Trigger Node:**  
   - Name: Hourly CRM Check  
   - Type: Schedule Trigger  
   - Configuration: Trigger every 1 hour, on workflow activation.

5. **Add Salesforce Node:**  
   - Name: Check Salesforce  
   - Type: Salesforce  
   - Configuration: Query opportunities where stage = "Contract Review" and lastModified within past hour.

6. **Add Merge Node:**  
   - Name: Merge Contract Sources  
   - Type: Merge (multiplex mode)  
   - Connect inputs from Monitor Contract Emails, Monitor Google Drive, and Check Salesforce nodes.

7. **Add Postgres Node:**  
   - Name: Check Duplicate  
   - Type: Postgres  
   - Configuration: SQL query to select contract_id from contracts where hash matches incoming fileHash.

8. **Add If Node:**  
   - Name: New Contract?  
   - Condition: Check if contract_id is empty or null.

9. **Add PDF Vector Extract Node:**  
   - Name: PDF Vector - Extract All Data  
   - Configuration: Set prompt for extracting detailed contract data, define JSON schema, input type file, and binaryPropertyName "data".

10. **Add PDF Vector Ask Node:**  
    - Name: PDF Vector - Risk Analysis  
    - Configuration: Set prompt for multi-dimensional risk scoring, input type file, binaryPropertyName "data".

11. **Add Postgres Node:**  
    - Name: Get Contract Template  
    - Configuration: Query active contract_templates table filtering by extracted contractType.

12. **Add Code Node:**  
    - Name: Analyze & Score Contract  
    - Configuration: Implement JavaScript code combining extracted data, risk analysis, template, and CRM data; calculate risk scores, approval levels, and alerts.

13. **Add Postgres Node:**  
    - Name: Save to Contract Repository  
    - Configuration: Insert detailed contract analysis into contracts table with all relevant fields and initial status "pending_approval".

14. **Add If Node:**  
    - Name: Needs Legal Review?  
    - Conditions: Risk level equals "High" OR missingClauses array length > 0.

15. **Add Slack Node:**  
    - Name: Notify Legal Team  
    - Configuration: Send message with contract details to #legal-contracts channel, including risk info and review URL.

16. **Add Salesforce Node:**  
    - Name: Update Salesforce  
    - Configuration: Update opportunity fields with contract ID, status "Under Review", expiration date, and risk score.

17. **Add Schedule Trigger Node:**  
    - Name: Daily Alert Check  
    - Configuration: Trigger once every day.

18. **Add Postgres Node:**  
    - Name: Get Upcoming Alerts  
    - Configuration: Query contracts with status "active" and alerts between current date and next 30 days.

19. **Add Slack Node:**  
    - Name: Send Daily Summary  
    - Configuration: Post daily summary of upcoming contract deadlines to "contract-alerts" Slack channel.

20. **Add Sticky Note:**  
    - Name: Analytics Dashboard  
    - Content: Static summary of total contracts, high risk, expiring soon, with dashboard URL.

21. **Connect Nodes According to Workflow:**  
    - Merge outputs to Check Duplicate → New Contract? → PDF Vector Extract → PDF Vector Risk Analysis + Get Contract Template → Analyze & Score Contract → Save to Contract Repository → Needs Legal Review? (true → Notify Legal Team) + Update Salesforce  
    - Schedule triggers connect to Check Salesforce and Get Upcoming Alerts → Send Daily Summary.

22. **Configure Credentials:**  
    - Email (IMAP credentials for contractreview@company.com)  
    - Google Drive (OAuth2 with access to contracts intake folder)  
    - Salesforce (OAuth2 for CRM API)  
    - Postgres (DB connection to contract repository and templates)  
    - Slack (OAuth2 with permissions for #legal-contracts and #contract-alerts channels)  
    - PDF Vector AI (API key or credentials for AI extraction nodes)

23. **Set Default Values and Constraints:**  
    - Poll intervals as per node configs  
    - Ensure contract intake folders and email addresses are correct  
    - Validate Salesforce filters and field mappings  
    - Verify JSON schema correctness in AI extract node

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow orchestrates end-to-end contract lifecycle with AI extraction, risk scoring, compliance, approval routing, and alerts. | Overview sticky note in workflow.                 |
| Contract Analytics Dashboard URL: https://analytics.company.com/contracts                                                        | Included in Analytics Dashboard sticky note.     |
| Slack channels used: #legal-contracts for legal alerts, #contract-alerts for deadline summaries                                   | Slack notification nodes.                         |
| Contract review system URL pattern: https://contracts.company.com/review/{{ contractId }}                                         | Used in Slack alert message links.                |
| AI extraction uses PDF Vector nodes with detailed JSON schemas for robust contract metadata extraction and risk analysis.        | In PDF Vector nodes configuration.                |
| Approval levels escalate by score and deal size: Manager → Director → VP → Legal                                                 | Explained in Analyze & Score Contract code node. |

---

**Disclaimer:** The provided content is based solely on an n8n automated workflow and complies with all applicable content policies. No illegal, offensive, or protected data are involved. All contract data processed is legal and public.