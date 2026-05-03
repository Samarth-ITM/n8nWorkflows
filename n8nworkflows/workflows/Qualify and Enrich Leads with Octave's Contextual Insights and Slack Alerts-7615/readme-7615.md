Qualify and Enrich Leads with Octave's Contextual Insights and Slack Alerts

https://n8nworkflows.xyz/workflows/qualify-and-enrich-leads-with-octave-s-contextual-insights-and-slack-alerts-7615


# Qualify and Enrich Leads with Octave's Contextual Insights and Slack Alerts

### 1. Workflow Overview

This workflow is designed to **qualify and enrich inbound sales leads** using Octave's AI-powered contextual insights, then deliver **contextual Slack alerts** to sales teams, Account Executives (AEs), and Revenue Operations (RevOps). Its purpose is to move beyond simple lead qualification, providing detailed understanding of a lead’s pain points, key responsibilities, value propositions, and references to enhance sales conversations.

The workflow is structured into five main logical blocks: 

- **1.1 Input Reception:** Captures inbound lead data via a webhook.
- **1.2 Lead Qualification:** Sends lead data to Octave’s qualification agent to score against the Ideal Customer Profile (ICP).
- **1.3 Filtering:** Filters out leads with low qualification scores.
- **1.4 Lead Enrichment:** For qualified leads, calls Octave’s enrichment agent to gather deep contextual insights.
- **1.5 Slack Notification:** Sends a detailed Slack alert with enriched lead information to a specified channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives inbound lead data through a configured webhook. It acts as the entry point into the workflow and captures raw lead details such as name, job title, company, domain, and LinkedIn profile URL.

- **Nodes Involved:**  
  - `Inbound Lead Webhook`

- **Node Details:**  
  - **Inbound Lead Webhook**  
    - Type: Webhook  
    - Role: Entry point to receive HTTP POST requests containing lead data.  
    - Configuration:  
      - `path`: Custom webhook path to receive leads (must be replaced with actual path).  
      - Method: Default POST assumed.  
      - No additional options configured.  
    - Key Expressions: None (raw JSON body expected).  
    - Input Connections: None (start node).  
    - Output Connections: Connects to `Qualify Lead with Octave`.  
    - Edge Cases:  
      - Failure if webhook path is not publicly accessible.  
      - Malformed or incomplete lead data could cause downstream failures.  
    - Sticky Note: “🚀 START HERE Replace webhook path. Configure lead source.”

---

#### 1.2 Lead Qualification

- **Overview:**  
  This block uses Octave’s AI to assess if the lead matches the ICP by scoring the lead’s attributes like job title, company domain, and LinkedIn profile.

- **Nodes Involved:**  
  - `Qualify Lead with Octave`

- **Node Details:**  
  - **Qualify Lead with Octave**  
    - Type: Octave node (custom integration).  
    - Role: Calls Octave’s qualification agent with lead data to get a qualification score.  
    - Configuration:  
      - `agentOId`: Octave qualification agent ID (must be replaced).  
      - Input parameters mapped from webhook JSON body fields: `jobTitle`, `firstName`, `companyName`, `companyDomain`, `profileURL`.  
      - Operation: `qualifyPerson`.  
    - Credentials: Octave API credentials required.  
    - Input: From `Inbound Lead Webhook`.  
    - Output: Qualification score and details passed to filter.  
    - Edge Cases:  
      - API authentication errors.  
      - Incorrect agent ID leading to failed qualification.  
      - Timeout or rate limits from Octave API.  
    - Sticky Note: “🎯 QUALIFICATION Octave scores vs ICP. Replace agent ID.”

---

#### 1.3 Filtering

- **Overview:**  
  This block removes leads with qualification scores below a configurable threshold to focus only on promising leads.

- **Nodes Involved:**  
  - `Filter Qualified Leads`

- **Node Details:**  
  - **Filter Qualified Leads**  
    - Type: Filter node.  
    - Role: Ensures only leads with score > 5 proceed.  
    - Configuration:  
      - Condition: Numeric comparison `$json.score > 5`.  
      - Case sensitive and strict type validation enabled.  
    - Input: From `Qualify Lead with Octave`.  
    - Output: Qualified leads flow to enrichment; others are dropped.  
    - Edge Cases:  
      - Missing or malformed `score` field causes filtering to fail.  
      - Threshold may need tuning to balance lead volume vs quality.  
    - Sticky Note: “🔍 FILTER Removes low scores. Adjust threshold (>5).”

---

#### 1.4 Lead Enrichment

- **Overview:**  
  For qualified leads, this block enriches data with deeper context such as primary responsibilities, pain points, value propositions, and references using Octave’s enrichment agent.

- **Nodes Involved:**  
  - `Enrich Lead with Context`

- **Node Details:**  
  - **Enrich Lead with Context**  
    - Type: Octave node.  
    - Role: Calls Octave’s enrichment agent to add contextual insights.  
    - Configuration:  
      - `agentOId`: Octave enrichment agent ID (must be replaced).  
      - Input parameters mapped similarly as qualification but uses data from the webhook node via expression referencing.  
      - Operation: `enrichPerson`.  
    - Credentials: Octave API credentials.  
    - Input: From `Filter Qualified Leads`.  
    - Output: Enriched lead data containing persona insights, pain points, value props, and references.  
    - Edge Cases:  
      - API errors or invalid enrichment agent ID.  
      - Missing linkedInProfile or company data may reduce enrichment quality.  
    - Sticky Note: “🔬 ENRICHMENT Extensive insights available. Explore output & customize.”

---

#### 1.5 Slack Notification

- **Overview:**  
  Sends a formatted Slack message containing enriched lead information to a specified Slack channel to alert sales teams with actionable insights.

- **Nodes Involved:**  
  - `Send Enriched Lead Alert`

- **Node Details:**  
  - **Send Enriched Lead Alert**  
    - Type: Slack node.  
    - Role: Posts a detailed message with enriched lead data to Slack channel.  
    - Configuration:  
      - `text`: Multi-line template using expressions to combine lead fields from webhook and enriched context JSON.  
      - `channelId`: Static channel ID (replaceable).  
      - Authentication: OAuth2 via Slack OAuth credentials.  
    - Input: From `Enrich Lead with Context`.  
    - Output: Slack message confirmation or error.  
    - Edge Cases:  
      - Slack OAuth token expiry or invalid permissions.  
      - Incorrect channel ID.  
      - Message length limits.  
    - Sticky Note: “📢 ENRICHED ALERTS Contextual lead insights. Update channel & format.”

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                     | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                  |
|---------------------------|------------------------|-----------------------------------|------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note - Main Overview| Sticky Note            | High-level workflow description   |                        |                           | 🎯 LEAD QUALIFICATION & ENRICHMENT FOR: Sales teams, AEs, RevOps...                          |
| Sticky Note - Webhook Setup| Sticky Note            | Setup instructions for webhook    |                        |                           | 🚀 START HERE Replace webhook path. Configure lead source.                                   |
| Sticky Note - Qualification| Sticky Note            | Setup instructions for Octave     |                        |                           | 🎯 QUALIFICATION Octave scores vs ICP. Replace agent ID.                                     |
| Sticky Note - Filter       | Sticky Note            | Setup instructions for filter     |                        |                           | 🔍 FILTER Removes low scores. Adjust threshold (>5).                                         |
| Sticky Note - Enrichment   | Sticky Note            | Setup instructions for enrichment |                        |                           | 🔬 ENRICHMENT Extensive insights available. Explore output & customize.                      |
| Sticky Note - Slack Alerts | Sticky Note            | Setup instructions for Slack node |                        |                           | 📢 ENRICHED ALERTS Contextual lead insights. Update channel & format.                        |
| Inbound Lead Webhook       | Webhook                | Receives inbound lead data        |                        | Qualify Lead with Octave   | 🚀 START HERE Replace webhook path. Configure lead source.                                   |
| Qualify Lead with Octave   | Octave                 | Lead qualification scoring        | Inbound Lead Webhook    | Filter Qualified Leads     | 🎯 QUALIFICATION Octave scores vs ICP. Replace agent ID.                                     |
| Filter Qualified Leads     | Filter                 | Filters leads by score threshold  | Qualify Lead with Octave| Enrich Lead with Context   | 🔍 FILTER Removes low scores. Adjust threshold (>5).                                         |
| Enrich Lead with Context   | Octave                 | Enriches lead with insights       | Filter Qualified Leads  | Send Enriched Lead Alert   | 🔬 ENRICHMENT Extensive insights available. Explore output & customize.                      |
| Send Enriched Lead Alert   | Slack                  | Sends Slack alert with enriched data| Enrich Lead with Context|                           | 📢 ENRICHED ALERTS Contextual lead insights. Update channel & format.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: `Inbound Lead Webhook`  
   - Type: Webhook  
   - Parameters:  
     - Path: Set your unique webhook path (e.g., `lead-inbound`)  
     - HTTP Method: POST (default)  
   - No credentials needed  
   - Purpose: Receive inbound lead JSON payloads

2. **Create an Octave Node for Qualification**  
   - Name: `Qualify Lead with Octave`  
   - Type: Octave Integration Node (custom)  
   - Parameters:  
     - `agentOId`: Insert your Octave qualification agent ID  
     - `operation`: Set to `qualifyPerson`  
     - Map input fields using expressions from webhook JSON body:  
       - `jobTitle`: `={{ $json.body.jobTitle }}`  
       - `firstName`: `={{ $json.body.firstName }}`  
       - `companyName`: `={{ $json.body.companyName }}`  
       - `companyDomain`: `={{ $json.body.companyDomain }}`  
       - `linkedInProfile`: `={{ $json.body.profileURL }}`  
   - Credentials: Octave API credentials (create and set up in n8n before use)  
   - Connect input from `Inbound Lead Webhook`

3. **Create a Filter Node**  
   - Name: `Filter Qualified Leads`  
   - Type: Filter  
   - Condition: Numeric filter where `$json.score > 5`  
   - Connect input from `Qualify Lead with Octave`  
   - Output path for leads passing the filter connects to enrichment node  

4. **Create an Octave Node for Enrichment**  
   - Name: `Enrich Lead with Context`  
   - Type: Octave Integration Node  
   - Parameters:  
     - `agentOId`: Insert your Octave enrichment agent ID  
     - `operation`: Set to `enrichPerson`  
     - Map input fields using expressions referencing `Inbound Lead Webhook` node:  
       - `jobTitle`: `={{ $('Inbound Lead Webhook').item.json.body.jobTitle }}`  
       - `firstName`: `={{ $('Inbound Lead Webhook').item.json.body.firstName }}`  
       - `companyName`: `={{ $('Inbound Lead Webhook').item.json.body.companyName }}`  
       - `companyDomain`: `={{ $('Inbound Lead Webhook').item.json.body.companyDomain }}`  
       - `linkedInProfile`: `={{ $('Inbound Lead Webhook').item.json.body.profileURL }}`  
   - Credentials: Octave API credentials  
   - Connect input from `Filter Qualified Leads`

5. **Create a Slack Node**  
   - Name: `Send Enriched Lead Alert`  
   - Type: Slack  
   - Parameters:  
     - `text`: Set multi-line Slack message template with variables, e.g.:  
       ```
       ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
       
       🔥 *Enriched Lead Alert*
       
       👤 *Name:* {{ $('Inbound Lead Webhook').item.json.body.firstName }} {{ $('Inbound Lead Webhook').item.json.body.lastName }}
       💼 *Title:* {{ $('Inbound Lead Webhook').item.json.body.jobTitle }}
       🏢 *Company:* {{ $('Inbound Lead Webhook').item.json.body.companyName }}
       🌐 *Domain:* {{ $('Inbound Lead Webhook').item.json.body.companyDomain }}
       🔗 *LinkedIn:* {{ $('Inbound Lead Webhook').item.json.body.profileURL }}
       
       🎯 *Key Responsibility:* {{ $json.persona.data.primaryResponsibilities[0] }}
       ⚡ *Main Pain Point:* {{ $json.persona.data.painPoints[0] }}
       💡 *Relevant Value Prop:* {{ $json.valueProps[0].details }}
       🏆 *Reference to Mention:* {{ $json.references[0].howTheyBenefitFromProduct }}
       ```
     - `channelId`: Set to your Slack channel ID (e.g., `C1234567890`)  
     - Authentication: Use OAuth2 with Slack OAuth credentials configured in n8n  
   - Connect input from `Enrich Lead with Context`

6. **Connect the nodes in sequence:**  
   `Inbound Lead Webhook` → `Qualify Lead with Octave` → `Filter Qualified Leads` → `Enrich Lead with Context` → `Send Enriched Lead Alert`

7. **Credential Setup:**  
   - Configure Octave API credentials with API key or token  
   - Configure Slack OAuth credentials with proper scopes to post messages to the selected channel

8. **Customizations:**  
   - Replace all placeholder IDs and paths (`webhook path`, `agentOId`s, Slack `channelId`) with your actual values.  
   - Adjust filter threshold as needed.  
   - Modify Slack message formatting to suit your team’s preference.  
   - Optionally, customize Octave agents or add additional fields for qualification/enrichment.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow targets Sales teams, AEs, and RevOps needing deeper lead insights beyond scoring.                       | Sticky Note - Main Overview                                                                      |
| Octave agents must be set up in your Octave account for qualification and enrichment with correct IDs.          | Official Octave documentation (not linked here)                                                 |
| Slack OAuth credentials require scopes for posting messages in channels.                                         | Slack API docs: https://api.slack.com/authentication/oauth-v2                                  |
| Webhook path must be externally accessible and secured as needed (e.g., via n8n cloud or self-hosting).          | n8n documentation: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                            |
| Consider error handling for API failures and Slack posting limits in production.                                | Best practices for API error handling in n8n workflows                                          |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow and complies strictly with content policies. It contains no illegal, offensive, or protected material. All data handled is legal and public.