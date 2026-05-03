Monitor zero-day threats with Anthropic Claude, Airtable, Slack and Jira

https://n8nworkflows.xyz/workflows/monitor-zero-day-threats-with-anthropic-claude--airtable--slack-and-jira-13692


# Monitor zero-day threats with Anthropic Claude, Airtable, Slack and Jira

This technical document provides a detailed breakdown of the **AI Zero-Day Threat Intelligence Monitor** workflow for n8n.

---

### 1. Workflow Overview
The **AI Zero-Day Threat Intelligence Monitor** is a proactive security automation designed to identify, analyze, and respond to emerging vulnerabilities (Zero-Days) before they are exploited. It bridges the gap between massive threat intelligence feeds and an organization’s specific infrastructure.

**Logical Phases:**
*   **1.1 Trigger & Asset Load:** Initiates the scan via schedule or webhook and retrieves the software/hardware inventory from Airtable.
*   **1.2 Multi-Source Intelligence Gathering:** Queries NVD, CISA KEV, GitHub, AlienVault OTX, and EPSS scores in parallel.
*   **1.3 Normalization & AI Scoring:** Deduplicates findings, correlates them with the asset inventory, and uses Claude AI (Anthropic) to perform a human-like risk assessment.
*   **1.4 Automated Response:** Executes multi-channel alerts (Slack, Email), opens Jira tickets for critical issues, triggers patch management webhooks, and logs results for compliance.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Asset Inventory Load
This block identifies which assets need monitoring and starts the execution.
*   **Nodes:** `On-Demand Scan Webhook`, `Hourly Threat Scan Schedule`, `Load Asset & Software Inventory`, `Build Scan Context & Search Terms`.
*   **Details:**
    *   **Airtable Node:** Searches for records where `Status = 'Active'` and `Monitor = TRUE`.
    *   **Build Scan Context (Code):** Normalizes Airtable data into a structured JSON array. It generates search terms (e.g., "Apache 2.4.54") for the threat feeds.
    *   **Edge Case:** If Airtable is empty, the Code node includes a "demoAssets" array to ensure the workflow doesn't fail silently during testing.

#### 2.2 Multi-Source Threat Intelligence Collection
A parallel data-gathering block that pulls from various public and private security databases.
*   **Nodes:** `Query NVD CVE Database`, `Fetch CISA Known Exploited Vulns`, `Query GitHub Security Advisories`, `Fetch AlienVault OTX Pulses`, `Fetch EPSS Exploit Probability Scores`.
*   **Details:**
    *   **NVD Node:** Uses a keyword search based on the asset inventory and filters for HIGH/CRITICAL severity.
    *   **GitHub Node:** Queries the `/advisories` endpoint for recent high-severity open-source vulnerabilities.
    *   **EPSS Node:** Pulls exploit prediction scores; CVEs with scores > 0.5 are prioritized.
    *   **OTX Node:** Checks AlienVault for community-reported "pulses" indicating active exploitation.

#### 2.3 Normalization & Claude AI Threat Scoring
The core intelligence engine where raw data is turned into actionable insights.
*   **Nodes:** `Merge All Threat Feed Results`, `Normalise, Deduplicate & Correlate`, `AI Threat Assessment & Prioritisation` (Agent), `Claude AI Model`, `Parse & Validate AI Assessment`.
*   **Details:**
    *   **Normalization (Code):** A complex script that deduplicates CVE IDs found across multiple sources and matches them against the specific versions of software found in the inventory.
    *   **AI Agent:** Uses the **Claude-3.5-Sonnet** model. It is programmed with a custom "Urgency Score Formula" (CVSS, CISA KEV status, internet exposure, and patch availability).
    *   **Input/Output:** Sends a JSON of correlated threats and assets; receives a structured JSON assessment containing executive summaries and technical remediation steps.

#### 2.4 Severity Routing & Automated Response
Routes findings to the appropriate teams and tools based on the AI-calculated threat level.
*   **Nodes:** `Filter Above Risk Threshold`, `Wait For Result`, `Route by Overall Threat Level`, `Alert SOC Team on Slack`, `Create Jira Threat Tickets`, `Submit Jira Issues via API`, `Send Threat Brief to Security Team`, `Trigger Patch Management System`, `Append to Threat Intelligence Log`.
*   **Details:**
    *   **Switch Node:** Branches logic into CRITICAL, HIGH, and MEDIUM/LOW paths.
    *   **Slack Node:** Routes CRITICAL alerts to `#soc-critical` and others to `#soc-alerts` using dynamic channel IDs.
    *   **Jira Code Node:** Transforms the AI assessment into a "Vulnerability" issue type with labels and priority mapping.
    *   **Email Node:** Generates a high-quality HTML brief for the security team with a summary of affected assets.
    *   **Patch Trigger:** Sends a POST request to a downstream system (e.g., Ansible, SCCM) to initiate automated patching.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On-Demand Scan Webhook | Webhook | Entry Point | - | Load Asset... | 1. Trigger & Asset Inventory Load |
| Hourly Threat Scan Schedule | Schedule | Entry Point | - | Load Asset... | 1. Trigger & Asset Inventory Load |
| Load Asset & Software Inventory | Airtable | Data Retrieval | Triggers | Build Scan Context | 1. Trigger & Asset Inventory Load |
| Build Scan Context... | Code | Data Prep | Load Asset... | Threat Feeds (5) | 1. Trigger & Asset Inventory Load |
| Query NVD CVE Database | HTTP Request | Intel Feed | Build Scan... | Merge | 2. Multi-Source Threat Intel Collection |
| Fetch CISA Known Exploited... | HTTP Request | Intel Feed | Build Scan... | Merge | 2. Multi-Source Threat Intel Collection |
| Query GitHub Security... | HTTP Request | Intel Feed | Build Scan... | Merge | 2. Multi-Source Threat Intel Collection |
| Fetch AlienVault OTX Pulses | HTTP Request | Intel Feed | Build Scan... | Merge | 2. Multi-Source Threat Intel Collection |
| Fetch EPSS Exploit... | HTTP Request | Intel Feed | Build Scan... | Merge | 2. Multi-Source Threat Intel Collection |
| Normalise, Deduplicate... | Code | Data Processing | Merge | AI Threat Assessment | 3. Normalisation · Correlation · AI |
| AI Threat Assessment... | AI Agent | Risk Analysis | Normalise... | Parse & Validate | 3. Normalisation · Correlation · AI |
| Claude AI Model | Anthropic Chat | AI Provider | - | AI Threat Assessment | 3. Normalisation · Correlation · AI |
| Parse & Validate... | Code | Post-AI Cleanup | AI Threat... | Filter | 3. Normalisation · Correlation · AI |
| Filter Above Risk Threshold | Filter | Logic Gate | Parse... | Wait For Result | 4. Severity Routing · SOC Alerts... |
| Route by Overall... | Switch | Logical Branch | Wait | Slack, Jira, Email, Patch | 4. Severity Routing · SOC Alerts... |
| Alert SOC Team on Slack | HTTP Request | Notification | Route... | Append to Log | 4. Severity Routing · SOC Alerts... |
| Create Jira Threat Tickets | Code | Ticket Logic | Route... | Submit Jira Issues | 4. Severity Routing · SOC Alerts... |
| Send Threat Brief... | Email Send | Notification | Route... | Trigger Patch | 4. Severity Routing · SOC Alerts... |
| Trigger Patch Management... | HTTP Request | Remediation | Email/Route | Append to Log | 4. Severity Routing · SOC Alerts... |
| Append to Threat Intel Log | Google Sheets | Logging | Various | Build SIEM Response | 4. Severity Routing · SOC Alerts... |
| Return Threat Intel... | Respond to Webhook | API Response | Build SIEM... | - | 4. Severity Routing · SOC Alerts... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Asset Setup:** Create an Airtable base with fields: `Asset ID`, `Hostname`, `IP Address`, `Software`, `Software Version`, `Internet Facing` (Checkbox), and `Monitor` (Checkbox).
2.  **Trigger Setup:**
    *   Add a **Schedule Trigger** set to `0 * * * *` (Hourly).
    *   Add a **Webhook** node (POST) for manual scans.
3.  **Credential Configuration:**
    *   **Anthropic:** Obtain an API key for Claude 3.5 Sonnet.
    *   **NVD:** Register for an API Key at NIST.gov.
    *   **GitHub:** Create a Personal Access Token (classic) with `read:packages`.
    *   **AlienVault:** Register for an OTX key.
    *   **Infrastructure:** Connect Slack OAuth2, Jira API Token, and Google Sheets Service Account.
4.  **Intel Feed Connections:**
    *   Connect the "Build Scan Context" Code node to 5 parallel HTTP Request nodes.
    *   Ensure all 5 feed nodes have `Continue on Fail` enabled so one down API doesn't stop the scan.
5.  **AI Integration:**
    *   Use the **AI Agent** (LangChain). Set the "System Message" to the provided security analyst prompt.
    *   Connect the **Anthropic Chat Model** node to the Agent.
6.  **Routing Logic:**
    *   Configure the **Switch Node** based on the expression `{{ $json.summary.overallThreatLevel }}`.
    *   Create 4 output branches: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`.
7.  **Ticket & Alert Setup:**
    *   For Jira: The node must use the "Vulnerability" issue type. Ensure the project key exists in your Jira instance.
    *   For Slack: Use the "Blocks" UI to format the JSON payload provided in the original workflow.
8.  **Logging:** Link the final nodes to a **Google Sheets** node using the "Append" operation to maintain a historical audit trail.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Project Credits** | Developed by [OneClick IT Solution](https://www.oneclickitsolution.com/contact-us/) |
| **API Rate Limits** | NVD and GitHub APIs have rate limits; ensure your n8n retry settings are configured. |
| **Security Context** | Claude AI requires a clear "System Message" to prevent hallucinations in vulnerability scores. |
| **Asset Matching** | The correlation logic uses substring matching; ensure software names in Airtable match common vendor naming. |