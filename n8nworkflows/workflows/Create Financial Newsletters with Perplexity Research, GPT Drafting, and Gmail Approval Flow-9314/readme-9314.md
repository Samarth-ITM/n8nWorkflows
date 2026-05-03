Create Financial Newsletters with Perplexity Research, GPT Drafting, and Gmail Approval Flow

https://n8nworkflows.xyz/workflows/create-financial-newsletters-with-perplexity-research--gpt-drafting--and-gmail-approval-flow-9314


# Create Financial Newsletters with Perplexity Research, GPT Drafting, and Gmail Approval Flow

---

# Create Financial Newsletters with Perplexity Research, GPT Drafting, and Gmail Approval Flow

---

## 1. Workflow Overview

This n8n workflow automates the creation, review, and approval process of weekly financial newsletters targeted at high-net-worth clients and family offices. It orchestrates data gathering from Perplexity AI research, drafts newsletter content with OpenAI GPT models, and manages an approval flow via Gmail and webhooks.

The workflow is logically structured into the following blocks:

- **1.1 Scheduled Trigger & Research Data Acquisition**: Periodic initiation and fetching of weekly financial market data from Perplexity API.
- **1.2 Research Data Parsing & Enrichment**: Cleaning, validating, normalizing, and enriching raw research JSON for editorial use.
- **1.3 Editorial Draft Generation**: Using GPT-4.1-mini to transform research data into a substantive client-ready newsletter draft.
- **1.4 HTML Rendering of Draft**: Using GPT-3.5-turbo to build a polished HTML newsletter from editorial JSON.
- **1.5 Quality Control & Compliance Checking**: Running a QC LLM that fact-checks and scans for compliance risks.
- **1.6 Email Preview & Approval Flow**: Sending preview emails with approve/revise links and hosting approval/revision webhooks for editors.
- **1.7 Final Draft Creation & Gmail Draft Sending**: Upon approval, generate the final Gmail draft for sending.
- **1.8 Revision Integration & Re-Drafting**: Handling revision notes, merging them, and re-running editorial drafting.

---

## 2. Block-by-Block Analysis

### 1.1 Scheduled Trigger & Research Data Acquisition

**Overview:**  
This block initiates the weekly newsletter generation on a fixed schedule and queries Perplexity’s API for detailed market research data in strict JSON format.

**Nodes Involved:**  
- Schedule Trigger  
- Week ranges + run_key  
- Research LLM  
- Parse Perplexity JSON  
- Enrich for Editorial  

#### Nodes

- **Schedule Trigger**  
  - Type: `n8n-nodes-base.scheduleTrigger`  
  - Purpose: Fires workflow weekly on Fridays at 8 AM (configurable).  
  - Configuration: Interval set to weekly, trigger hour 8.  
  - Inputs: None (trigger node).  
  - Outputs: Triggers next node "Week ranges + run_key".  
  - Edge Cases: Misfire if n8n server down; scheduling misconfiguration.

- **Week ranges + run_key**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Calculates ISO date ranges for last week (Monday to Sunday) and next week, and generates a stable run key for idempotency.  
  - Key Logic: Uses JavaScript Date; hashes lastMon and lastSun ISO strings to create a 12-char run_key.  
  - Outputs: week_label, start_iso, end_iso, next_start_iso, next_end_iso, run_key.  
  - Edge Cases: Timezone issues if n8n server timezone changes; hashing stability critical.

- **Research LLM**  
  - Type: `n8n-nodes-base.perplexity` (Perplexity AI API)  
  - Purpose: Sends system and user prompts to Perplexity to get a strict JSON research summary of markets for the defined week.  
  - Configuration: TopP=1, maxTokens=4000, temperature=0.2; detailed prompt instructing Perplexity to return ONE valid JSON object with comprehensive financial data, sources, and coverage.  
  - Inputs: week_label, start_iso, end_iso from previous node.  
  - Outputs: Raw JSON response with research data.  
  - Edge Cases: API rate limits, malformed JSON response, network errors.  
  - Sticky Note: "Substitute LLM instructions on what and how to research the topic of interest."

- **Parse Perplexity JSON**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Robustly parses the Perplexity JSON output, repairing common formatting issues such as code fences, dangling commas, or partial strings.  
  - Key Expressions: Custom JS parser with regex-based cleaning, fallback reparsing, and normalization of fields to arrays/objects.  
  - Inputs: Raw JSON string from "Research LLM".  
  - Outputs: Parsed, normalized research JSON.  
  - Edge Cases: Invalid JSON that cannot be repaired triggers error with raw content snippet.

- **Enrich for Editorial**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Adds editorial metadata to the research JSON such as firm name, contact URL, view URL (dynamic with runKey), and current year.  
  - Inputs: Parsed research JSON and week label/run_key from "Week ranges + run_key".  
  - Outputs: Enriched JSON passed to editorial drafting.  
  - Sticky Note: Requires personalization of firm info and URLs.

---

### 1.2 Editorial Draft Generation

**Overview:**  
Transforms enriched market research JSON into a client-focused newsletter draft JSON with detailed sections and compliance guardrails.

**Nodes Involved:**  
- Editor LLM  

#### Nodes

- **Editor LLM**  
  - Type: `@n8n/n8n-nodes-langchain.openAi` (OpenAI GPT-4.1-mini)  
  - Purpose: Converts enriched research JSON into a comprehensive newsletter JSON including headline, sections, subject options, metadata, and signature.  
  - Configuration: MaxTokens=4500, temperature=0.6; detailed system prompt specifying audience, tone, newsletter structure (7 sections), content requirements, compliance rules, and output schema.  
  - Inputs: Enriched JSON from previous node, plus optionally prior draft and revision notes for edits.  
  - Outputs: JSON newsletter content with editorial narrative and structure.  
  - Edge Cases: Token limits, prompt injection risks, incomplete JSON output.  
  - Sticky Note: Editorial prompts can be personalized for tone and style.

---

### 1.3 HTML Rendering of Draft

**Overview:**  
Converts the editorial JSON draft into a styled, ready-to-send HTML email newsletter.

**Nodes Involved:**  
- HTML Builder LLM  
- QC LLM  
- Assemble Preview (Code)  
- Send Preview  

#### Nodes

- **HTML Builder LLM**  
  - Type: `@n8n/n8n-nodes-langchain.openAi` (OpenAI GPT-3.5-turbo)  
  - Purpose: Generates full HTML document for the newsletter using strict rendering rules (headings, typography, color palette, table formatting).  
  - Inputs: Editorial draft JSON from "Editor LLM".  
  - Outputs: JSON with `subject` and full HTML string.  
  - Edge Cases: HTML injection risks mitigated by prompt instructions; token limits.  
  - Sticky Note: Output styling and branding should be aligned with client guidelines.

- **QC LLM**  
  - Type: `@n8n/n8n-nodes-langchain.openAi` (OpenAI GPT-3.5-turbo)  
  - Purpose: Performs quality control by fact-checking the draft against research JSON and scanning for compliance issues such as promissory language or forbidden claims.  
  - Configuration: MaxTokens=1000, temperature=0, deterministic output with specified JSON schema.  
  - Inputs: Draft HTML, research JSON, and week context.  
  - Outputs: JSON object listing fact issues, compliance flags, and notes.  
  - Edge Cases: False positives/negatives in fact-checking; compliance policy adjustments possible.

- **Assemble Preview (Code)**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Aggregates HTML, subject, warnings, editor JSON, and research into a single payload; encodes context in base64url for URL-safe passing to approval links.  
  - Outputs: Object containing email subject, HTML preview, warnings, and encoded context.  
  - Edge Cases: Encoding errors, missing HTML or subject cause error.

- **Send Preview**  
  - Type: `n8n-nodes-base.gmail`  
  - Purpose: Sends an email preview of the newsletter to a specified email address with approve and revise action buttons linking to approval webhooks.  
  - Configuration: Uses Gmail OAuth2 credentials; subject prefixed with “[Preview]”; buttons embed encoded context in URLs.  
  - Inputs: Payload from "Assemble Preview (Code)".  
  - Outputs: None (send action).  
  - Sticky Note: Requires personalization of sender email and domain placeholders.

---

### 1.4 Email Preview & Approval Flow (Webhooks)

**Overview:**  
Handles editor interaction via webhooks for newsletter approval or revision requests, including rendering approval/revision forms and processing submissions.

**Nodes Involved:**  
- Approval Webhook (GET)  
- Respond to Webhook (HTML page)  
- Approval Submit (POST)  
- Decode  
- Normalize  
- Check Action Type  
- If  
- Merge Revision Notes  
- Respond Form HTML1  
- Create Final Draft for Editor  
- Respond to Webhook (text)  

#### Nodes

- **Approval Webhook (GET)**  
  - Type: `n8n-nodes-base.webhook` (HTTP GET)  
  - Purpose: Serves the approval/revision HTML page with form, populated dynamically with run key and action from query parameters.  
  - Outputs: HTML page allowing editor to approve or send revision notes.  
  - Sticky Note: Requires domain placeholder update.

- **Respond to Webhook (HTML page)**  
  - Type: `n8n-nodes-base.respondToWebhook`  
  - Purpose: Renders the actual HTML form for revision notes or auto-submits if approving.  
  - Inputs: Query parameters for run, action, and context.  
  - Outputs: HTML page with form or auto-submit script.

- **Approval Submit (POST)**  
  - Type: `n8n-nodes-base.webhook` (HTTP POST)  
  - Purpose: Receives submission of approval or revision notes from the form; triggers downstream processing.  
  - Outputs: Passes payload to "Decode" node.

- **Decode**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Decodes base64url-encoded context payload received from webhook, normalizes fields, extracts action and notes.  
  - Outputs: JSON normalized with action, subject, html, research, editor_json, week_meta, notes.

- **Normalize**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Further normalizes incoming data, cleans subject line, adjusts for approve action.  
  - Outputs: Normalized JSON for condition check.

- **Check Action Type**  
  - Type: `n8n-nodes-base.if`  
  - Purpose: Branches workflow based on whether the action is "approve" or not.  
  - Output 1 (true): Approval path → "Create Final Draft for Editor".  
  - Output 2 (false): Revision path → "If" node.

- **If**  
  - Type: `n8n-nodes-base.if`  
  - Purpose: Checks if revision notes are empty or not.  
  - Output 1 (true): Has notes → "Merge Revision Notes".  
  - Output 2 (false): No notes → "Respond Form HTML1" (re-display form).

- **Merge Revision Notes**  
  - Type: `n8n-nodes-base.code`  
  - Purpose: Appends new revision notes to revision history, increments revision number, merges context for re-drafting.  
  - Outputs: Merged JSON with revision metadata and notes.

- **Respond Form HTML1**  
  - Type: `n8n-nodes-base.respondToWebhook`  
  - Purpose: Returns a minimal HTML form for revision notes submission, supports auto-submit on approve.  
  - Inputs: Query parameters.  
  - Outputs: HTML form page.

- **Create Final Draft for Editor**  
  - Type: `n8n-nodes-base.gmail`  
  - Purpose: Sends the approved final newsletter as a Gmail draft to the configured account for final sending.  
  - Configuration: Uses Gmail OAuth2; creates draft with HTML body and subject line.  
  - Inputs: Final approved HTML and subject from "Normalize" node.  
  - Outputs: None (send draft action).

- **Respond to Webhook (text)**  
  - Type: `n8n-nodes-base.respondToWebhook`  
  - Purpose: Sends a simple text response (HTTP 200) confirming receipt of approval or revision submission.  
  - Outputs: HTTP response.

---

### 1.5 Summary Notes on Sticky Notes

- Several nodes have sticky notes emphasizing the need for personalization: updating sender email addresses, OAuth2 Gmail credentials, and domain placeholders such as `[YOURDOMAIN]`.
- Editorial prompt nodes allow light adaptation to match client tone and branding.
- Schedule trigger timing can be adjusted to preferred publishing schedules.

---

## 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                          | Input Node(s)              | Output Node(s)                    | Sticky Note                                                                                   |
|---------------------------|-------------------------------|----------------------------------------|----------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger          | scheduleTrigger               | Initiates workflow weekly               | -                          | Week ranges + run_key            | Schedule Trigger: Adjust interval and triggerAtHour to preferred publishing time               |
| Week ranges + run_key     | code                         | Calculates date ranges and run key      | Schedule Trigger            | Research LLM                    |                                                                                                |
| Research LLM              | perplexity                   | Fetches weekly market research JSON     | Week ranges + run_key       | Parse Perplexity JSON           | Substitute LLM instructions on what and how to research the topic                             |
| Parse Perplexity JSON     | code                         | Robust JSON parsing and normalization   | Research LLM                | Enrich for Editorial            |                                                                                                |
| Enrich for Editorial      | code                         | Adds editorial metadata (firm, URLs)    | Parse Perplexity JSON       | Editor LLM                     | Personalize firm_name, contact_url, view_url, signature block                                  |
| Editor LLM                | openAi (GPT-4.1-mini)        | Generates client newsletter JSON draft  | Enrich for Editorial, Merge Revision Notes | HTML Builder LLM              | Editorial prompts and output styling can be personalized                                       |
| HTML Builder LLM          | openAi (GPT-3.5-turbo)       | Converts JSON draft to HTML newsletter  | Editor LLM                 | QC LLM                        | Personalize colors, typography, and branding                                                   |
| QC LLM                    | openAi (GPT-3.5-turbo)       | Fact-checks and compliance scans draft  | HTML Builder LLM            | Assemble Preview (Code)         | Compliance rules can be adjusted                                                              |
| Assemble Preview (Code)   | code                         | Combines data and encodes context       | QC LLM                     | Send Preview                   |                                                                                                |
| Send Preview              | gmail                        | Sends preview email with approve/revise | Assemble Preview (Code)     | -                            | Update sender address and OAuth2 credentials; update domain placeholders                       |
| Approval Webhook (GET)    | webhook (GET)                | Serves approval/revision HTML form      | -                          | Respond to Webhook (HTML page)  | Update domain placeholders                                                                    |
| Respond to Webhook (HTML page) | respondToWebhook          | Renders approval/revision form           | Approval Webhook (GET)      | -                            |                                                                                                |
| Approval Submit (POST)    | webhook (POST)               | Receives approval or revision submission | -                          | Decode                        |                                                                                                |
| Decode                    | code                         | Decodes context and normalizes data     | Approval Submit (POST)      | Normalize                     |                                                                                                |
| Normalize                 | code                         | Cleans and prepares data for branching  | Decode                     | Check Action Type              |                                                                                                |
| Check Action Type         | if                           | Branches on approve or revise action    | Normalize                  | Create Final Draft for Editor, If |                                                                                                |
| If                       | if                           | Checks if revision notes are empty      | Check Action Type           | Merge Revision Notes, Respond Form HTML1 |                                                                                                |
| Merge Revision Notes      | code                         | Merges new revision notes to history    | If                         | Editor LLM                    |                                                                                                |
| Respond Form HTML1        | respondToWebhook             | Serves minimal revision notes form      | If                         | -                            |                                                                                                |
| Create Final Draft for Editor | gmail                      | Sends final approved draft as Gmail draft | Check Action Type (approve) | Respond to Webhook             | Requires Gmail OAuth2 credentials                                                             |
| Respond to Webhook (text) | respondToWebhook             | Sends confirmation HTTP 200 text response | Create Final Draft for Editor | -                            |                                                                                                |
| Sticky Note               | stickyNote                   | Notes for personalization and setup     | -                          | -                            | Multiple nodes covered by sticky notes for personalization and domain/email updates           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" Node**  
   - Type: scheduleTrigger  
   - Parameters: Set interval to weekly; trigger at 8 AM (adjust as needed).  
   - Connect output to next node.

2. **Create "Week ranges + run_key" Node**  
   - Type: code  
   - Paste JavaScript code to calculate last week’s Monday-Sunday range and next week’s date range; generate SHA256-based run_key.  
   - Connect from Schedule Trigger.

3. **Create "Research LLM" Node**  
   - Type: perplexity (Perplexity API node)  
   - Configure credentials with Perplexity API key.  
   - Paste system prompt instructing to output strict JSON market research with detailed coverage, referencing week ranges from previous node.  
   - Set options: topP=1, maxTokens=4000, temperature=0.2.  
   - Connect from Week ranges + run_key.

4. **Create "Parse Perplexity JSON" Node**  
   - Type: code  
   - Paste robust JS parser code for Perplexity JSON output to handle formatting irregularities and normalize arrays/objects.  
   - Connect from Research LLM.

5. **Create "Enrich for Editorial" Node**  
   - Type: code  
   - Paste JS code that adds editorial metadata: firm_name, contact_url, view_url (with run_key), and current year.  
   - Customize firm_name, contact_url, and domain in view_url to your organization.  
   - Connect from Parse Perplexity JSON.

6. **Create "Editor LLM" Node**  
   - Type: OpenAI (Langchain node)  
   - Configure OpenAI API credentials (GPT-4.1-mini).  
   - Paste system prompt specifying detailed newsletter drafting instructions including tone, audience, structure, compliance guardrails, and output schema.  
   - Set maxTokens=4500, temperature=0.6.  
   - Input from Enrich for Editorial and also from Merge Revision Notes (for revision loop).  
   - Connect from Enrich for Editorial and Merge Revision Notes.

7. **Create "HTML Builder LLM" Node**  
   - Type: OpenAI (Langchain node)  
   - Use GPT-3.5-turbo with maxTokens=4000, temperature=0.2.  
   - Paste system prompt instructing strict HTML output generation with styling and table rules.  
   - Connect from Editor LLM.

8. **Create "QC LLM" Node**  
   - Type: OpenAI (Langchain node)  
   - Use GPT-3.5-turbo, maxTokens=1000, temperature=0, deterministic output.  
   - Paste system prompt for fact-checking and compliance scanning with strict JSON output schema.  
   - Connect from HTML Builder LLM.

9. **Create "Assemble Preview (Code)" Node**  
   - Type: code  
   - Paste provided JS code that extracts subject, HTML, warnings, encodes context as base64url.  
   - Connect from QC LLM.

10. **Create "Send Preview" Node**  
    - Type: gmail  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient email to your test or distribution list.  
    - Set subject to "[Preview] <subject>".  
    - Set message body to include HTML preview plus approve/revise buttons with encoded context URLs.  
    - Replace `[YOURDOMAIN]` with your n8n domain or self-hosted domain.  
    - Connect from Assemble Preview (Code).

11. **Create "Approval Webhook (GET)" Node**  
    - Type: webhook (GET)  
    - Path: "newsletter-approval"  
    - Connect to "Respond to Webhook (HTML page)".

12. **Create "Respond to Webhook (HTML page)" Node**  
    - Type: respondToWebhook  
    - Paste HTML+JS that renders approval/revision forms, auto-submits on approve.  
    - Connect from Approval Webhook (GET).

13. **Create "Approval Submit (POST)" Node**  
    - Type: webhook (POST)  
    - Path: "newsletter-approval-submit"  
    - Connect to "Decode".

14. **Create "Decode" Node**  
    - Type: code  
    - Paste JS code to decode base64url context and normalize inputs.  
    - Connect from Approval Submit (POST).

15. **Create "Normalize" Node**  
    - Type: code  
    - Paste JS code to clean subject line, adjust for approval action.  
    - Connect from Decode.

16. **Create "Check Action Type" Node**  
    - Type: IF  
    - Condition: if `action` equals "approve" (case-insensitive).  
    - True branch: Connect to "Create Final Draft for Editor".  
    - False branch: Connect to "If".

17. **Create "If" Node**  
    - Type: IF  
    - Condition: notes field is not empty (revision notes present).  
    - True branch: Connect to "Merge Revision Notes".  
    - False branch: Connect to "Respond Form HTML1".

18. **Create "Merge Revision Notes" Node**  
    - Type: code  
    - Paste JS code that appends notes to revision history, increments revision number, merges context.  
    - Connect to "Editor LLM" (revision loop).

19. **Create "Respond Form HTML1" Node**  
    - Type: respondToWebhook  
    - Serves minimal revision form with hidden fields and auto-submit logic.  
    - Connect from "If" node (false branch).

20. **Create "Create Final Draft for Editor" Node**  
    - Type: gmail  
    - Configure Gmail OAuth2 credentials.  
    - Setup to create a Gmail draft (not send) with approved newsletter HTML and subject.  
    - Connect from Check Action Type (true branch).

21. **Create "Respond to Webhook" Node**  
    - Type: respondToWebhook  
    - Sends HTTP 200 plain text confirmation.  
    - Connect from Create Final Draft for Editor.

22. **Add Sticky Note Nodes**  
    - Add sticky notes on critical nodes reminding to:  
      - Personalize sender email (Send Preview, Create Final Draft)  
      - Replace `[YOURDOMAIN]` placeholders with your domain  
      - Customize editorial prompts and styling  
      - Adjust scheduling as needed  
      - Update firm/company info in enrichment node

---

## 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                   |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Replace `[YOURDOMAIN]` placeholders in URLs with your actual n8n cloud or self-hosted domain                    | Send Preview node, Approval Webhook links                        |
| Configure Gmail OAuth2 credentials with your Google account for Send Preview and Create Final Draft nodes       | Gmail nodes                                                      |
| Editorial LLM and HTML Builder LLM system prompts can be lightly customized to match your organization's tone  | Editor LLM node, HTML Builder LLM node                           |
| Compliance rules in QC LLM node can be adjusted to fit jurisdictional requirements                              | QC LLM node                                                     |
| Schedule Trigger timing is customizable to preferred publishing schedule                                        | Schedule Trigger node                                            |
| Firm/company info (name, contact URL, website URL) should be updated in "Enrich for Editorial" node             | Enrich for Editorial node                                        |
| For detailed prompt examples and best practices, see: https://docs.n8n.io/integrations/builtin/nodes/OpenAI/     | Official n8n OpenAI node documentation                           |

---

**Disclaimer:**  
The provided analysis and documentation are based solely on an automated n8n workflow export. All data processed are public and legal. No content violates platform policies.

---