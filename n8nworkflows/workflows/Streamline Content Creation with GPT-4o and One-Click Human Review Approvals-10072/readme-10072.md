Streamline Content Creation with GPT-4o and One-Click Human Review Approvals

https://n8nworkflows.xyz/workflows/streamline-content-creation-with-gpt-4o-and-one-click-human-review-approvals-10072


# Streamline Content Creation with GPT-4o and One-Click Human Review Approvals

---

### 1. Workflow Overview

This workflow automates the process of AI-assisted content creation from initial request to human review and final approval. It targets marketing and content teams who want to streamline content generation using OpenAI’s GPT-4o model, combined with a structured human review and one-click approval system. The workflow ensures content drafts are generated quickly by AI, sent to reviewers with interactive approval links, and logged systematically in Google Sheets for transparency and tracking.

**Logical blocks:**

- **1.1 Content Request Input**: Receives content requirements via a customizable web form.
- **1.2 AI Content Generation**: Uses GPT-4o to generate a draft based on input parameters.
- **1.3 Content Processing & Scoring**: Analyzes AI output, extracts metadata, and scores content quality.
- **1.4 Logging & Tracking**: Logs requests and updates drafts in Google Sheets.
- **1.5 Review Link Generation & Notification**: Creates actionable review links and emails the content to reviewers.
- **1.6 Review Action Handling**: Captures reviewer decisions (approve/edit/reject) through webhook endpoints and updates tracking accordingly.
- **1.7 Response to Reviewer**: Sends styled HTML responses confirming the reviewer’s action.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Request Input

- **Overview:** Presents a web form to collect comprehensive content requests, including type, topic, audience, tone, word count, reviewer email, and priority.
- **Nodes Involved:** `📥 Content Request Form`, `Prepare Request Data`, `Log to Tracking Sheet`
- **Node Details:**

  - **📥 Content Request Form**
    - Type: `formTrigger` (Webhook Trigger for form submissions)
    - Configuration: Path set to a unique webhook URL; form fields include dropdowns (content type, tone, priority), text inputs (topic, audience, key points), and email field for reviewer.
    - Key Expressions: Uses standard form field bindings; required fields ensure essential data.
    - Output: JSON with all form inputs.
    - Failure Modes: Missing required fields, webhook connectivity issues.

  - **Prepare Request Data**
    - Type: `Set` node to enrich form data.
    - Configuration: Assigns unique request ID (`REQ-yyyyMMdd-HHmmss`), timestamp of creation, initial status (`pending_generation`), and translates word count dropdown into numeric target (Short=400, Medium=1000, Long=2000).
    - Input: JSON from form.
    - Output: JSON with enriched metadata.
    - Failure Modes: Date formatting errors, expression evaluation issues.

  - **Log to Tracking Sheet**
    - Type: `Google Sheets` node (append operation)
    - Configuration: Appends a new row with all request details to the “Content Requests” sheet.
    - Credentials: Requires configured Google Sheets OAuth2 credentials.
    - Input: Prepared request data.
    - Output: Confirmation of append.
    - Failure Modes: Google Sheets API errors, authentication failures.

---

#### 2.2 AI Content Generation

- **Overview:** Generates the initial content draft using OpenAI GPT-4o based on the request parameters.
- **Nodes Involved:** `🤖 Generate AI Content`, `OpenAI GPT-4o`
- **Node Details:**

  - **🤖 Generate AI Content**
    - Type: `chainLlm` (LangChain LLM node)
    - Configuration: Defines a prompt template tailored to the content type, topic, audience, tone, word count, and key points. The prompt instructs the AI to produce structured content with title, meta description (if blog post), main content, and CTA.
    - Inputs: Request data JSON.
    - Outputs: Prompt text to language model node.
    - Failure Modes: Expression errors if variables missing.

  - **OpenAI GPT-4o**
    - Type: `lmChatOpenAi` (OpenAI Chat API node)
    - Configuration: Uses GPT-4o model, max tokens 3000, temperature 0.7 for creativity.
    - Credentials: Requires OpenAI API key with GPT-4o access.
    - Input: Prompt from previous node.
    - Output: AI-generated content text.
    - Failure Modes: API rate limits, invalid credentials, timeouts.

---

#### 2.3 Content Processing & Scoring

- **Overview:** Extracts structured output from AI text, calculates word count, title, and a heuristic quality score based on length, formatting, CTAs, and repetition.
- **Nodes Involved:** `Process Generated Content`
- **Node Details:**

  - **Process Generated Content**
    - Type: `Code` node (JavaScript)
    - Configuration:
      - Extracts generated text from AI response.
      - Calculates word count and reading time (words/200 wpm).
      - Parses the first line as title.
      - Applies heuristic scoring considering length accuracy, presence of subheadings, bullet points, CTAs, and sentence diversity.
      - Sets status to `pending_review`.
    - Input: AI-generated content JSON.
    - Output: Enriched JSON with `aiDraft`, `title`, `stats` (wordCount, qualityScore, etc.), and updated status.
    - Edge Cases: If AI output is empty or malformed, scoring may fail or produce misleading results.

---

#### 2.4 Logging & Tracking

- **Overview:** Updates the Google Sheet entry with the AI draft, quality score, word count, and generation timestamp.
- **Nodes Involved:** `Update Sheet with Draft`
- **Node Details:**

  - **Update Sheet with Draft**
    - Type: `Google Sheets` node (update operation)
    - Configuration: Updates the existing row in the “Content Requests” sheet identified by request ID with AI draft and stats.
    - Credentials: Google Sheets OAuth2.
    - Input: Processed content data.
    - Output: Confirmation of update.
    - Failure Modes: Sheet row identification errors, API failures.

---

#### 2.5 Review Link Generation & Notification

- **Overview:** Generates direct approve, reject, and edit links for reviewers and emails the draft with these actionable buttons.
- **Nodes Involved:** `🔗 Generate Review Links`, `📧 Create Review Email`, `✉️ Send Review Request`
- **Node Details:**

  - **🔗 Generate Review Links**
    - Type: `Code` node
    - Configuration: Constructs URLs for review actions (approve/edit/reject) based on environment variable `WEBHOOK_URL` and request ID.
    - Input: Updated content JSON.
    - Output: JSON extended with `reviewLinks` object.
    - Failure Modes: Missing environment variable, malformed URLs.

  - **📧 Create Review Email**
    - Type: `Code` node
    - Configuration: Builds a styled HTML email including content metadata, quality score visualization, AI draft preview, and three buttons linking to review actions.
    - Input: JSON with content and review links.
    - Output: JSON with `htmlEmail` field containing full HTML string.
    - Edge Cases: Large drafts may cause email overflow; HTML rendering varies by client.

  - **✉️ Send Review Request**
    - Type: `Gmail` node (send email)
    - Configuration: Sends the generated HTML email to the reviewer’s email with dynamic subject containing content title.
    - Credentials: OAuth2 Gmail account.
    - Output: Email send confirmation.
    - Failure Modes: SMTP authentication issues, invalid email, quota limits.

---

#### 2.6 Review Action Handling

- **Overview:** Receives reviewer action via webhook, routes based on action type, updates Google Sheet status accordingly.
- **Nodes Involved:** `🔔 Review Action Webhook`, `Check: Approve?`, `Check: Reject?`, `✅ Update: Approved`, `✏️ Update: Needs Edit`, `❌ Update: Rejected`
- **Node Details:**

  - **🔔 Review Action Webhook**
    - Type: `Webhook` node
    - Configuration: Listens on a unique path for query parameters `action` and `id`.
    - Output: JSON with query parameters.
    - Failure Modes: Missing parameters, invalid webhook calls.

  - **Check: Approve?**
    - Type: `If` node
    - Checks if `action` query equals “approve” (case insensitive).
    - Routes to approval update or else to next check.

  - **Check: Reject?**
    - Type: `If` node
    - Checks if `action` equals “reject”.
    - Routes to reject update or else to edit request.

  - **✅ Update: Approved**
    - Type: `Google Sheets` node (update)
    - Updates row with status “approved”, timestamp `Approved At`, final content from AI draft, and review status “Approved”.

  - **✏️ Update: Needs Edit**
    - Type: `Google Sheets` node (update)
    - Updates row with status “revision_requested”, review status “Needs Changes”, and timestamp for revision request.

  - **❌ Update: Rejected**
    - Type: `Google Sheets` node (update)
    - Updates row with status “rejected”, review status “Rejected”, and rejection timestamp.

- All update nodes use the same Google Sheet and require OAuth2 credentials.

- Failure Modes: Invalid request IDs, Google Sheets update conflicts, webhook call errors.

---

#### 2.7 Response to Reviewer

- **Overview:** Sends back a styled HTML confirmation page to the reviewer based on their action.
- **Nodes Involved:** `Respond: Approved`, `Respond: Edit Requested`, `Respond: Rejected`
- **Node Details:**

  - Type: `Respond to Webhook` nodes
  - Configuration: Each returns a fully styled HTML page reflecting the outcome (approval, edit request, or rejection) with request ID shown.
  - Input: Query parameters from webhook node.
  - Output: HTTP response to browser.
  - Edge Cases: Browser compatibility for styling; no fallback if node fails.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                     | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                          |
|----------------------------|----------------------------------|-----------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| 📥 Content Request Form     | formTrigger                      | Receive content request via form  | -                                | Prepare Request Data             | ### Workflow Stages 1. Submit a form with topic, tone, and keywords. 2. GPT-4o generates content and assigns a quality score (0–100). 3. Reviewer receives an email to approve, edit, or reject the draft. 4. All review actions are automatically logged in Google Sheets with timestamps and approval status.  |
| Prepare Request Data        | Set                              | Enrich request data with ID, time | 📥 Content Request Form           | Log to Tracking Sheet            |                                                                                                    |
| Log to Tracking Sheet       | Google Sheets                    | Log initial request in sheet      | Prepare Request Data              | 🤖 Generate AI Content           |                                                                                                    |
| 🤖 Generate AI Content       | LangChain LLM (chainLlm)         | Create AI content prompt          | Log to Tracking Sheet             | OpenAI GPT-4o                   |                                                                                                    |
| OpenAI GPT-4o              | LangChain OpenAI Chat            | Generate AI content draft         | 🤖 Generate AI Content            | Process Generated Content        | ## Requirements n8n v1.0+ instance, OpenAI API key with GPT-4o access, and Google Workspace with OAuth2 credentials. Custom options Choose model: gpt-4o, 4o-mini, or 3.5-turbo. Adjust score weights: Readability 40%, Keywords 30%, Length 30%. Key benefits Generate drafts 99% faster and approve content 95% quicker. Centralized tracking ensures clarity and accountability.  |
| Process Generated Content   | Code                             | Extract content data & quality score | OpenAI GPT-4o                   | Update Sheet with Draft          |                                                                                                    |
| Update Sheet with Draft     | Google Sheets                    | Update sheet with draft and stats | Process Generated Content        | 🔗 Generate Review Links         |                                                                                                    |
| 🔗 Generate Review Links    | Code                             | Generate approve/edit/reject URLs | Update Sheet with Draft          | 📧 Create Review Email           |                                                                                                    |
| 📧 Create Review Email      | Code                             | Build HTML email for reviewer     | 🔗 Generate Review Links          | ✉️ Send Review Request           |                                                                                                    |
| ✉️ Send Review Request      | Gmail                           | Send review email to reviewer     | 📧 Create Review Email           | -                               |                                                                                                    |
| 🔔 Review Action Webhook    | Webhook                         | Receive review action from links  | -                                | Check: Approve?, Check: Reject? |                                                                                                    |
| Check: Approve?             | If                              | Route if action=approve           | 🔔 Review Action Webhook          | ✅ Update: Approved, ✏️ Update: Needs Edit |                                                                                                    |
| Check: Reject?              | If                              | Route if action=reject            | 🔔 Review Action Webhook          | ❌ Update: Rejected             |                                                                                                    |
| ✅ Update: Approved         | Google Sheets                   | Update sheet on approval          | Check: Approve?                  | Respond: Approved               |                                                                                                    |
| ✏️ Update: Needs Edit       | Google Sheets                   | Update sheet on edit request      | Check: Approve?                  | Respond: Edit Requested         |                                                                                                    |
| ❌ Update: Rejected         | Google Sheets                   | Update sheet on rejection         | Check: Reject?                   | Respond: Rejected               |                                                                                                    |
| Respond: Approved           | Respond to Webhook              | Confirm approval to reviewer      | ✅ Update: Approved              | -                               |                                                                                                    |
| Respond: Edit Requested     | Respond to Webhook              | Confirm edit request to reviewer  | ✏️ Update: Needs Edit            | -                               |                                                                                                    |
| Respond: Rejected           | Respond to Webhook              | Confirm rejection to reviewer     | ❌ Update: Rejected              | -                               |                                                                                                    |
| Sticky Note                | Sticky Note                    | Workflow overview and steps       | -                                | -                               | ### Workflow Stages 1. Submit a form with topic, tone, and keywords. 2. GPT-4o generates content and assigns a quality score (0–100). 3. Reviewer receives an email to approve, edit, or reject the draft. 4. All review actions are automatically logged in Google Sheets with timestamps and approval status.  |
| Sticky Note1               | Sticky Note                    | Setup instructions and how it works | -                              | -                               | ## How it works Submit a form with your topic, tone, and keywords. GPT-4o generates the content and assigns a quality score (0–100). The reviewer receives an email to approve, edit, or reject the draft—all actions are automatically logged in Google Sheets for tracking and audit purposes. ## Setup steps 1. Import the workflow JSON file into n8n 2. Connect your OpenAI and Google account credentials 3. Update three variables in the workflow: SHEET_ID (your Google Sheets document ID), REVIEWER_EMAIL (recipient for review notifications), and WEBHOOK_URL (for form submissions) 4. Test the workflow with a sample submission |
| Sticky Note2               | Sticky Note                    | Workflow overview summary         | -                                | -                               | ## Overview Automate AI content creation from request to approval. While AI writes quickly, human review often delays delivery—and multiple tools create workflow gaps and version confusion. This unified solution streamlines the entire process, enabling teams to produce quality content at scale with transparent tracking. |
| Sticky Note3               | Sticky Note                    | Requirements and benefits         | -                                | -                               | ## Requirements n8n v1.0+ instance, OpenAI API key with GPT-4o access, and Google Workspace with OAuth2 credentials. ## Custom options Choose model: gpt-4o, 4o-mini, or 3.5-turbo. Adjust score weights: Readability 40%, Keywords 30%, Length 30%. ## Key benefits Generate drafts 99% faster and approve content 95% quicker. Centralized tracking ensures clarity and accountability. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Content Request Form Trigger**
   - Add a `formTrigger` node.
   - Set webhook path (e.g., unique UUID).
   - Configure form fields:
     - Content Type (dropdown with 8 options)
     - Topic / Title (text, required)
     - Target Audience (text, required)
     - Key Points / Requirements (textarea, required)
     - Tone of Voice (dropdown, 6 options)
     - Word Count (dropdown, 3 options)
     - Reviewer Email (email, required)
     - Priority (dropdown, 3 options)
   - Title and description as per workflow.
   
2. **Add a Set Node to Prepare Request Data**
   - Connect from form trigger.
   - Create fields:
     - `requestId`: `"REQ-" + current date/time formatted as yyyyMMdd-HHmmss`
     - `createdAt`: current timestamp ISO string
     - `status`: `"pending_generation"`
     - `wordTarget`: Number mapped from `Word Count` dropdown (Short=400, Medium=1000, Long=2000)
   
3. **Add Google Sheets Node to Log Initial Request**
   - Operation: Append
   - Document: Your Google Sheet ID
   - Sheet Name: "Content Requests"
   - Map columns: Content Type, Topic, Status, AI Draft (empty), Audience, Priority, Reviewer, Created At, Request ID, Word Count, Published At (empty), Final Content (empty), Review Status (empty)
   - Use OAuth2 credentials for Google Sheets.
   - Connect from Prepare Request Data.
   
4. **Add LangChain Chain LLM Node for AI Content Prompt**
   - Connect from Google Sheets append node.
   - Configure prompt template per workflow text, incorporating variables from input JSON.
   - Use "define" prompt type.

5. **Add OpenAI GPT-4o Node**
   - Connect from LangChain node.
   - Set model to `gpt-4o`.
   - Set max tokens: 3000, temperature: 0.7.
   - Attach OpenAI API credentials with GPT-4o access.

6. **Add Code Node to Process Generated Content**
   - Connect from OpenAI GPT-4o.
   - Implement JavaScript code to:
     - Extract generated text.
     - Calculate word count, title, reading time.
     - Calculate heuristic quality score.
     - Set status `pending_review`, add stats object.
   
7. **Add Google Sheets Node to Update Sheet with Draft**
   - Operation: Update
   - Document & Sheet same as initial log.
   - Map status, AI Draft, word count, generated at timestamp, quality score columns.
   - Connect from Process Generated Content.

8. **Add Code Node to Generate Review Links**
   - Connect from update draft node.
   - Use environment variable `WEBHOOK_URL`.
   - Create URLs for approve, reject, and edit actions with requestId as query param.

9. **Add Code Node to Create Review Email**
   - Connect from generate review links node.
   - Build HTML email with styling, metadata, quality score visualization, draft preview, and buttons linking to review URLs.

10. **Add Gmail Node to Send Review Email**
    - Connect from create review email node.
    - Set recipient as reviewer email.
    - Subject includes content title.
    - Use Gmail OAuth2 credentials.

11. **Add Webhook Node to Receive Review Actions**
    - Set webhook path (unique UUID).
    - Response mode: Respond node.
    - Outputs to two IF nodes.

12. **Add If Node to Check Approve Action**
    - Condition: `action` query equals "approve" (case insensitive).
    - True branch to Approved update node; False branch to Reject check.

13. **Add If Node to Check Reject Action**
    - Condition: `action` equals "reject".
    - True branch to Rejected update node; False branch to Edit requested update node.

14. **Add Google Sheets Update Nodes for Approval, Edit Request, Rejection**
    - All update “Content Requests” sheet using request ID.
    - Approval node sets status `approved`, timestamps, final content, review status.
    - Edit node sets status `revision_requested`, review status `Needs Changes`, timestamp.
    - Reject node sets status `rejected`, review status `Rejected`, timestamp.
    - Use Google Sheets OAuth2 credentials.

15. **Add Respond to Webhook Nodes for Each Review Outcome**
    - Each node sends styled HTML confirmation to reviewer browser.
    - Include request ID and relevant messaging.
    - Connect each update node to corresponding respond node.

16. **(Optional) Add Sticky Notes**
    - Document workflow stages, instructions, overview, and requirements for maintainers and users.

17. **Set Environment Variables**
    - `WEBHOOK_URL`: Base URL of your n8n instance accessible externally.
    - `YOUR_GOOGLE_SHEET_ID`: Your Google Sheet document ID.
    - Configure OpenAI and Google OAuth2 credentials accordingly.

18. **Activate and Test**
    - Test by submitting the form, checking AI generation, receiving email, and performing approve/edit/reject actions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates AI content creation with integrated human review and approval, reducing content production delays and avoiding version confusion.                                                                                                                                                                                                            | Workflow overview sticky notes                                                                                                      |
| Requires n8n version 1.0+ with access to OpenAI GPT-4o API (or compatible GPT-4o variants), Google Workspace with OAuth2 credentials for Google Sheets and Gmail.                                                                                                                                                                                                | Sticky note with requirements                                                                                                       |
| Customizable scoring weights for readability, keyword inclusion, and length can be adjusted by modifying the heuristic in the Process Generated Content node.                                                                                                                                                                                                  | Recommend reviewing and tuning code node for quality score calculation                                                               |
| Review emails include instant action buttons linking back to n8n webhook endpoints, enabling one-click approval workflows.                                                                                                                                                                                                                                      | HTML email construction in Create Review Email node                                                                                  |
| Google Sheets tracks all stages: request, generation, review decisions, timestamps, and final content for audit and team transparency.                                                                                                                                                                                                                        | Multiple Google Sheets nodes with append/update operations                                                                           |
| For detailed setup instructions, ensure environment variables and credentials are correctly configured, and test webhook URLs for external access.                                                                                                                                                                                                           | Sticky note with setup instructions                                                                                                 |
| Additional customization ideas: add Slack or Teams notifications, integrate CMS publishing APIs, or extend with version control.                                                                                                                                                                                                                                | Possible workflow extensions                                                                                                        |

---

**Disclaimer:** The text above originates exclusively from an automated workflow created with n8n, respecting all current content policies. No illegal, offensive, or protected content is included. All data processed is legal and public.

---