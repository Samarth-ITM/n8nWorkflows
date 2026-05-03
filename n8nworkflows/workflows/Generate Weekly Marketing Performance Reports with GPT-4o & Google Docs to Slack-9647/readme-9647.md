Generate Weekly Marketing Performance Reports with GPT-4o & Google Docs to Slack

https://n8nworkflows.xyz/workflows/generate-weekly-marketing-performance-reports-with-gpt-4o---google-docs-to-slack-9647


# Generate Weekly Marketing Performance Reports with GPT-4o & Google Docs to Slack

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly marketing performance reports using simulated ad data and AI-generated insights. It is designed for marketing teams and agencies to streamline reporting, replacing manual data collection and formatting with an end-to-end automated process.

The workflow logically divides into these blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow weekly on a fixed schedule.
- **1.2 Metrics Generation (Demo Data):** Produces reproducible, fake ad campaign performance metrics for multiple channels (Google Ads, Meta, TikTok, YouTube) for demonstration purposes.
- **1.3 AI Executive Summary:** Sends the generated metrics to an OpenAI GPT-4o model to create a concise, actionable executive summary.
- **1.4 Markdown Report Assembly:** Combines raw metrics and AI summary into a well-structured Markdown report including tables for totals, channel breakdowns, and top campaigns by ROAS.
- **1.5 Google Docs Report Creation and Update:** Creates a new Google Doc for the report, then inserts the Markdown content into the document.
- **1.6 Slack Notification:** Posts a message in a Slack channel with topline metrics and a direct link to the Google Doc report.
- **1.7 Documentation and Recommendations:** Sticky notes provide explanations, value propositions, and optional enhancements for using templates.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow automatically every 7 days at 7 AM, initiating the weekly reporting cycle.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - Type: Schedule Trigger (n8n built-in)  
  - Configuration: Interval set to every 7 days, triggering at 7:00 AM.  
  - Input: None (trigger node)  
  - Output: Connected to "Google Ads Demo" node.  
  - Version: 1.2  
  - Edge Cases: Misconfiguration could cause missed or multiple triggers; time zone considerations may affect scheduling.  
  - Notes: Reliable and simple trigger for periodic execution.

---

#### 1.2 Metrics Generation (Demo Data)

- **Overview:**  
  Generates reproducible dummy performance metrics for multiple ad channels and campaigns over the last 7 days, simulating real data without external API calls.

- **Nodes Involved:**  
  - Google Ads Demo (Code node)  

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Produces fake, yet structured and plausible ad metrics for Google Ads, Meta, TikTok, and YouTube.  
  - Configuration:  
    - Uses a seeded random number generator for reproducibility.  
    - Defines channels with base impressions, CPC, conversion rates.  
    - Generates campaign-level data with impressions, clicks, conversions, spend, revenue, ROAS, CTR, and conversion rates.  
    - Outputs JSON with period, summary totals, and detailed by-channel data.  
  - Inputs: Triggered from Schedule Trigger.  
  - Outputs: JSON metrics passed to "Message a model".  
  - Version: 2  
  - Edge Cases: No external dependencies; errors only possible if code is modified incorrectly.  
  - Notes: Replace this node with real API nodes to integrate live data.

---

#### 1.3 AI Executive Summary

- **Overview:**  
  Sends the generated metrics JSON to OpenAI GPT-4o-mini model to create a short executive summary highlighting key wins, issues, and recommendations.

- **Nodes Involved:**  
  - Message a model (OpenAI node)

- **Node Details:**  
  - Type: OpenAI (Langchain)  
  - Role: Calls GPT-4o-mini to generate textual summary from input data.  
  - Configuration:  
    - Model ID: gpt-4o-mini  
    - Messages: System prompt sets role as senior performance marketer; user prompt formats input JSON data and requests 120–180 word summary.  
  - Input: Metrics JSON from "Google Ads Demo".  
  - Output: AI-generated summary text.  
  - Credentials: Requires OpenAI API credentials.  
  - Version: 1.8  
  - Edge Cases: Possible failure due to API limits, authentication errors, or malformed input JSON; network timeouts.  
  - Notes: Critical node for adding AI-driven insights.

---

#### 1.4 Markdown Report Assembly

- **Overview:**  
  Constructs a comprehensive Markdown report combining raw metrics and the AI executive summary, including tables for totals, channel performance, and top campaigns by ROAS.

- **Nodes Involved:**  
  - Code in JavaScript

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Formats and combines data from prior nodes into a Markdown string.  
  - Configuration:  
    - Reads JSON from "Google Ads Demo" and "Message a model".  
    - Helper functions format numbers, generate Markdown tables for channels and campaigns.  
    - Report includes header, period, executive summary, totals list, channel table, top campaign table, and footer note.  
  - Input: Metrics JSON and AI summary JSON.  
  - Output: JSON with Markdown report string and data summary.  
  - Version: 2  
  - Edge Cases: Errors if referenced node names ("Google Ads Demo", "Message a model") are changed or missing JSON data; null or undefined values handled gracefully.  
  - Notes: Node names must exactly match those referenced in code.

---

#### 1.5 Google Docs Report Creation and Update

- **Overview:**  
  Creates a new Google Docs document titled with the reporting period, then updates this document by inserting the Markdown report content.

- **Nodes Involved:**  
  - Create a document (Google Docs node)  
  - Update a document (Google Docs node, update operation)

- **Node Details:**  
  - Create a document:  
    - Type: Google Docs  
    - Role: Creates a new document in a specified Google Drive folder.  
    - Configuration:  
      - Title template uses report period start and end dates.  
      - Folder ID set to target directory.  
    - Credentials: Requires Google Docs OAuth2 credentials.  
    - Input: Markdown report JSON from "Code in JavaScript".  
    - Output: Document metadata including document ID.  
    - Version: 2  
    - Edge Cases: Auth errors, folder permission issues, API rate limits.  
  - Update a document:  
    - Type: Google Docs  
    - Role: Inserts Markdown text into the created document.  
    - Configuration:  
      - Operation: Update  
      - Action: Insert text (Markdown report) at start or end.  
      - Document URL dynamically set from created document ID.  
    - Credentials: Same as above.  
    - Input: Document ID from "Create a document" and report content from "Code in JavaScript".  
    - Output: Updated document metadata.  
    - Version: 2  
    - Edge Cases: Document lock or permission errors, malformed Markdown insertion, API failures.  
  - Notes: An optional sticky note recommends using Google Docs templates by copying and updating for branded reports.

---

#### 1.6 Slack Notification

- **Overview:**  
  Sends a Slack message with the report summary and a direct link to the Google Doc, notifying the team that the weekly report is ready.

- **Nodes Involved:**  
  - Send a message (Slack node)

- **Node Details:**  
  - Type: Slack  
  - Role: Posts a formatted message in a specific Slack channel.  
  - Configuration:  
    - Message text includes emojis, report period, topline ROAS and spend, and Google Doc link.  
    - Channel ID is configurable and selected from a list.  
    - Auth uses OAuth2.  
  - Input: Document metadata from "Update a document", report summary from "Code in JavaScript".  
  - Credentials: Slack OAuth2 credentials required.  
  - Version: 2.3  
  - Edge Cases: Authentication failures, invalid channel ID, Slack API rate limits, network errors.  
  - Notes: Ensures team awareness and easy access to reports.

---

#### 1.7 Documentation and Recommendations (Sticky Notes)

- **Overview:**  
  Provides inline documentation, value propositions, and suggestions for workflow enhancement.

- **Nodes Involved:**  
  - Multiple Sticky Note nodes positioned near functional nodes.

- **Node Details:**  
  - Types: Sticky Notes (n8n standard)  
  - Roles: Explain each block’s purpose, value, configuration tips, and enhancements like Google Docs templates.  
  - Content Highlights:  
    - Workflow purpose and benefits (time-saving, standardization, AI insights).  
    - Demo data generation explanation.  
    - AI summary usage and OpenAI credential reminder.  
    - Markdown report building instructions with node name dependencies.  
    - Google Docs OAuth and document creation notes.  
    - Slack notification setup instructions.  
    - Optional enhancement: Use Google Docs template copy for polished branding.  
  - Version: 1  
  - Input/Output: None (documentation only).  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                         | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                                             |
|--------------------|-------------------------------|---------------------------------------|----------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger               | Initiates workflow every 7 days at 7AM| None                 | Google Ads Demo          |                                                                                                                                         |
| Google Ads Demo     | Code                          | Generates reproducible fake ad metrics| Schedule Trigger     | Message a model          | ## 📊 Generate Metrics (Demo)\nProduces fake ad performance data for Google Ads, Meta, TikTok & YouTube.\n\n👉 Replace with real API connectors |
| Message a model     | OpenAI (Langchain openAi)      | Creates AI executive summary           | Google Ads Demo       | Code in JavaScript       | ## 🤖 AI Executive Summary\nSends metrics to OpenAI (LLM) to create a concise summary with wins, issues, and recommendations.\n\n👉 Make sure your **OpenAI credentials** are connected. |
| Code in JavaScript  | Code                          | Builds Markdown report combining metrics & AI summary | Message a model       | Create a document        | ## 📝 Build Markdown Report\nCombines raw metrics + AI summary into a structured Markdown report.\nIncludes totals, per-channel table, and top campaigns by ROAS.\n\n👉 Node name references must match exactly (“Google Ads Demo”, “Message a model”). |
| Create a document   | Google Docs                   | Creates new Google Doc for report      | Code in JavaScript    | Update a document        | ## 📄 Google Docs Creation\nCreates a new Google Doc titled  \n**“Weekly Performance Report – [Start Date] to [End Date]”**.\n\n👉 Requires Google Docs OAuth connection. |
| Update a document   | Google Docs                   | Inserts Markdown into created Doc      | Create a document     | Send a message           | ## 🖊️ Update Google Doc\nInserts the Markdown report into the created document.\n👉 You’ll get a polished report ready for sharing.        |
| Send a message      | Slack                         | Sends Slack notification with report link | Update a document     | None                     | ## 💬 Slack Notification\nSends a Slack message with topline numbers (ROAS, Spend) + direct link to the Google Doc.\n👉 Connect your Slack account and set the channel ID. |
| Sticky Note         | Sticky Note                   | Descriptive documentation node         | None                 | None                     | ## 📊 What the Automation Does\n- Runs automatically on a schedule (e.g. every Monday).  \n- Pulls campaign performance data (demo).\n- Uses AI to write summary.\n- Builds Markdown report.\n- Creates Google Doc.\n- Notifies Slack.\n- Emails report. |
| Sticky Note1        | Sticky Note                   | Explains workflow value                 | None                 | None                     | ## 💡 Why This Is Valuable\n- Saves time.\n- Standardizes reporting.\n- Adds AI insights.\n- Improves transparency.\n- Scales easily.\n- Professional client experience. |
| Sticky Note2        | Sticky Note                   | Explains demo metrics generation        | None                 | None                     | ## 📊 Generate Metrics (Demo)\nProduces fake ad performance data for Google Ads, Meta, TikTok & YouTube.\n\n👉 Replace with real API connectors (Google Ads, Meta Ads, TikTok, YouTube) if you want live data. |
| Sticky Note3        | Sticky Note                   | Explains AI summary node usage          | None                 | None                     | ## 🤖 AI Executive Summary\nSends metrics to OpenAI (LLM) to create a concise summary with wins, issues, and recommendations.  \n\n👉 Make sure your **OpenAI credentials** are connected. |
| Sticky Note4        | Sticky Note                   | Explains Markdown report assembly       | None                 | None                     | ## 📝 Build Markdown Report\nCombines raw metrics + AI summary into a structured Markdown report.\nIncludes totals, per-channel table, and top campaigns by ROAS.\n\n👉 Node name references must match exactly (“Google Ads Demo”, “Message a model”). |
| Sticky Note5        | Sticky Note                   | Explains Google Docs creation            | None                 | None                     | ## 📄 Google Docs Creation\nCreates a new Google Doc titled  \n**“Weekly Performance Report – [Start Date] to [End Date]”**.\n\n👉 Requires Google Docs OAuth connection. |
| Sticky Note6        | Sticky Note                   | Explains Slack notification              | None                 | None                     | ## 💬 Slack Notification\nSends a Slack message with topline numbers (ROAS, Spend) + direct link to the Google Doc.  \n👉 Connect your Slack account and set the channel ID. |
| Sticky Note7        | Sticky Note                   | Explains Google Docs updating            | None                 | None                     | ## 🖊️ Update Google Doc\nInserts the Markdown report into the created document.  \n👉 You’ll get a polished report ready for sharing.       |
| Sticky Note8        | Sticky Note                   | Recommends Google Docs template usage    | None                 | None                     | ## 💡 Extra Recommendation: Use a Google Docs Template\nInstead of creating a blank Google Doc, connect this workflow to a pre-styled Google Docs template.\n👉 This makes your reports look more professional right away.\n\nBenefits: consistent branding, polished design, easier reading.\n\nHow: create template, copy document, update with report content. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Reporting Automation").

2. **Add a Schedule Trigger node**:  
   - Set the interval to every 7 days.  
   - Set the trigger time to 7:00 AM.

3. **Add a Code node named `Google Ads Demo`:**  
   - Paste the provided JavaScript code that generates reproducible fake metrics for Google Ads, Meta, TikTok, and YouTube campaigns over the last 7 days.  
   - This node outputs JSON structured with `period`, `summary`, and `byChannel`.

4. **Connect Schedule Trigger → Google Ads Demo.**

5. **Add an OpenAI node named `Message a model`:**  
   - Select the Langchain OpenAI node type.  
   - Set model to `gpt-4o-mini`.  
   - Configure the messages:  
     - System role: "You are a senior performance marketer. Write concise, actionable summaries."  
     - User content: Template prompt that includes JSON insertion of metrics (`period`, `summary`, first channel data), requesting a 120–180 word executive summary with wins, issues, recommendations.  
   - Link OpenAI credentials (API key).  
   - Connect Google Ads Demo → Message a model.

6. **Add a Code node named `Code in JavaScript`:**  
   - Insert the JavaScript that fetches JSON from `Google Ads Demo` and `Message a model` nodes.  
   - Format a Markdown report with sections: header, period, executive summary, totals, channel table, top campaigns table, and footer.  
   - Output JSON with `report` (Markdown string), `period`, `summary`, and `byChannel`.  
   - Connect Message a model → Code in JavaScript.

7. **Add a Google Docs node named `Create a document`:**  
   - Operation: Create document.  
   - Title: `Weekly Performance Report – {{ $json.period.start }} to {{ $json.period.end }}` (use expression).  
   - Set `folderId` to target Google Drive folder.  
   - Link Google Docs OAuth2 credentials.  
   - Connect Code in JavaScript → Create a document.

8. **Add a Google Docs node named `Update a document`:**  
   - Operation: Update document.  
   - Action: Insert text (Markdown report).  
   - Document URL: Use expression to get created document ID (`={{ $json.id }}`).  
   - Text to insert: `={{ $('Code in JavaScript').item.json.report }}`.  
   - Use same Google Docs OAuth2 credentials.  
   - Connect Create a document → Update a document.

9. **Add a Slack node named `Send a message`:**  
   - Authentication: OAuth2 with Slack.  
   - Channel: Select channel ID (e.g., from dropdown or enter manually).  
   - Message text template:  
     ```
     :bar_chart: Weekly report is ready
     Period: {{ $('Code in JavaScript').item.json.period.start }} → {{ $('Code in JavaScript').item.json.period.end }}
     Topline: ROAS {{ $('Code in JavaScript').item.json.summary.roas }} | Spend ${{ $('Code in JavaScript').item.json.summary.spend }}

     Open Doc: https://docs.google.com/document/d/{{ $json.documentId }}/edit
     ```  
   - Connect Update a document → Send a message.

10. **Add Sticky Notes** at appropriate positions with the described content to document and explain workflow sections for future maintainers.

11. **Optional enhancement:**  
    - Instead of "Create a document", implement a "Copy Document" node to duplicate a pre-styled Google Docs template.  
    - Update the copied document with the Markdown report.  
    - This preserves branding and design.

12. **Test the workflow:**  
    - Manually trigger or wait for scheduled run.  
    - Verify generated metrics, AI summary quality, Google Doc creation and content, Slack notification delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates weekly marketing report generation with demo data, AI summary, Google Docs, and Slack notifications.                   | Workflow description and purpose                                                                                                 |
| Replace demo metrics node with real API integrations (Google Ads API, Meta Ads API, TikTok Ads API, YouTube Analytics API) for live data. | Sticky Note2 content                                                                                                             |
| Ensure OpenAI and Google Docs OAuth2 credentials are properly configured and have required scopes for API calls.                         | Sticky Note3 and Sticky Note5 content                                                                                            |
| Slack OAuth2 connection must have permissions to post in the configured channel.                                                          | Sticky Note6 content                                                                                                             |
| Use Google Docs templates and copy them to maintain consistent branding and design for reports.                                           | Sticky Note8 content                                                                                                             |
| AI summary uses GPT-4o-mini model with Langchain node for best summarization.                                                             | Node "Message a model" configuration                                                                                             |
| Markdown report includes summary tables and top campaigns sorted by ROAS, providing actionable insights.                                  | Node "Code in JavaScript" explanation                                                                                           |
| Scheduling at 7 AM every 7 days ensures reports are generated weekly, typically before team review meetings.                              | Schedule Trigger configuration                                                                                                  |

---

This document fully describes the automated workflow "Generate Weekly Marketing Performance Reports with GPT-4o & Google Docs to Slack" and enables reproduction, understanding, and troubleshooting by advanced users or AI agents.