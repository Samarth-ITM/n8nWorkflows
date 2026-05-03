Daily Jira Ticket Summarizer using GPT-5 and Jira API

https://n8nworkflows.xyz/workflows/daily-jira-ticket-summarizer-using-gpt-5-and-jira-api-8103


# Daily Jira Ticket Summarizer using GPT-5 and Jira API

---

### 1. Workflow Overview

This workflow automates the daily summarization of Jira tickets created within a specified project using OpenAI's GPT-5 model and the Jira API. It targets teams and managers who need concise, professional summaries of daily Jira issues, including ticket details, comments, and proposed solutions, delivered automatically via email.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Initialization:** Starts workflow manually or on a schedule and sets the Jira project key.
- **1.2 Ticket Retrieval:** Queries Jira API to fetch all tickets created today from the specified project.
- **1.3 Ticket Processing Loop:** Iterates through each ticket to extract details and retrieve all related comments.
- **1.4 AI Summarization:** Uses GPT-5 via LangChain nodes to analyze combined ticket data and comments, generating structured summaries and proposed solutions.
- **1.5 Aggregation & Formatting:** Collects individual ticket summaries, formats them into a readable email body with ticket links.
- **1.6 Notification Delivery:** Sends the compiled daily ticket summary report by Gmail to designated recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Initialization

**Overview:**  
This block initiates the workflow either manually or on a schedule and sets the Jira project key for filtering tickets.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  
- Set Project Key  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for testing or ad hoc runs.  
  - Configuration: Default; no parameters.  
  - Inputs: None  
  - Outputs: To Set Project Key node  
  - Failure Modes: None typical; manual operation dependent on user.  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow automatically on a defined recurring interval (default is every minute/hour/day based on configuration).  
  - Configuration: Interval set to default (can be customized).  
  - Inputs: None  
  - Outputs: To Set Project Key node  
  - Failure Modes: Scheduling misconfiguration may cause missed triggers.  

- **Set Project Key**  
  - Type: Set Node  
  - Role: Defines the Jira project key ("SUP") used by subsequent Jira API calls to filter tickets.  
  - Configuration: Assigns string value `SUP` to variable `Project Key`.  
  - Expressions: None dynamic, static assignment.  
  - Inputs: From trigger nodes  
  - Outputs: To Get All Tickets node  
  - Failure Modes: Hardcoded project key requires manual update for other projects.  

---

#### 2.2 Ticket Retrieval

**Overview:**  
Fetches all Jira tickets created today for the specified project using JQL filters.

**Nodes Involved:**  
- Get All Tickets  

**Node Details:**

- **Get All Tickets**  
  - Type: Jira Node (getAll operation)  
  - Role: Retrieves all tickets where project equals `Project Key` and creation date is today.  
  - Configuration:  
    - JQL: `project = {{ $json['Project Key'] }} AND created >= startOfDay()`  
    - Return All: True (all tickets matching query fetched)  
    - Limit: Not set (unlimited due to returnAll)  
  - Inputs: From Set Project Key  
  - Outputs: To Split Out node  
  - Credentials: Jira Software Cloud API (configured with valid OAuth or API token)  
  - Failure Modes:  
    - Jira API rate limiting or authentication failure  
    - Invalid JQL syntax or project key errors  
    - Empty ticket results if no tickets created today  

---

#### 2.3 Ticket Processing Loop

**Overview:**  
Splits the batch of tickets into individual items, then iteratively fetches detailed comments for each ticket and prepares data for AI processing.

**Nodes Involved:**  
- Split Out  
- Loop Tickets  
- Get Comments from Ticket  
- Merge  
- Merge5  

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Extracts key fields (ticket key, summary, description) from the bulk ticket list into individual items for processing.  
  - Configuration:  
    - Field to Split Out: `key` (ticket key)  
    - Fields to Include: `fields.description, fields.summary`  
  - Inputs: From Get All Tickets  
  - Outputs: To Loop Tickets  
  - Failure Modes: Missing fields in tickets may cause incomplete data.  

- **Loop Tickets**  
  - Type: Split In Batches  
  - Role: Processes tickets one by one (batch size = 1 by default) to ensure discrete AI summarization.  
  - Configuration: Default batch size (assumed 1)  
  - Inputs: From Split Out and from Set Output (loop back for aggregation)  
  - Outputs: Two paths:  
    1. To Aggregate node (collect processed tickets)  
    2. To Get Comments from Ticket and Merge nodes (for each ticket processing)  
  - Failure Modes: Large ticket volume may slow processing; batch size should be tuned accordingly.  

- **Get Comments from Ticket**  
  - Type: Jira Node (issueComment getAll operation)  
  - Role: Retrieves all comments for the current ticket key.  
  - Configuration:  
    - Issue Key: `={{ $json.key }}` (dynamic from current ticket)  
    - Return All: True (all comments fetched)  
  - Inputs: From Loop Tickets  
  - Outputs: To Merge node  
  - Credentials: Jira Software Cloud API  
  - Failure Modes:  
    - No comments for ticket (returns empty)  
    - API rate limits or auth errors  

- **Merge**  
  - Type: Merge  
  - Role: Combines ticket data and comments into a single JSON object for AI summarization.  
  - Configuration: Mode set to "Combine All" (merges inputs)  
  - Inputs:  
    - Primary: Ticket data (from Loop Tickets)  
    - Secondary: Comments (from Get Comments from Ticket)  
  - Outputs: To Merge5 node  
  - Failure Modes: Mismatched data may cause merge errors or incomplete summaries.  

- **Merge5**  
  - Type: Merge  
  - Role: Receives output from AI summarization and merges with input data for output structuring.  
  - Configuration: Mode "Combine All"  
  - Inputs:  
    - Primary: AI Summarizer output  
    - Secondary: Merged ticket and comments data  
  - Outputs: To Set Output  
  - Failure Modes: Merge conflicts or missing fields if AI output is incomplete.  

---

#### 2.4 AI Summarization

**Overview:**  
Uses OpenAI GPT-5 via LangChain integration to generate professional, structured ticket summaries and proposed solutions based on combined ticket details and comments.

**Nodes Involved:**  
- Ticket Summarizer  
- OpenAI Chat Model  
- Structured Output Parser  

**Node Details:**

- **Ticket Summarizer**  
  - Type: LangChain Chain LLM Node  
  - Role: Sends prompt with ticket data to OpenAI GPT-5 model and parses output using a JSON schema.  
  - Configuration:  
    - Prompt includes ticket title, description, comments (serialized JSON), and instructions to generate JSON output with `"Ticket Summary"` and `"Proposed Solution"` fields.  
    - Output Parser enabled with JSON schema to ensure structured response.  
  - Inputs: From Merge node (ticket+comments combined)  
  - Outputs: To Merge5 node  
  - Expressions: Uses templated prompt with embedded variables (e.g., `{{ $json['fields.summary'] }}`)  
  - Failure Modes:  
    - API quota exceeded or authentication failure  
    - Incomplete or malformed AI responses causing parse errors  
    - Timeout or network issues with OpenAI API  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model Node  
  - Role: Provides GPT-5 model for AI summarization.  
  - Configuration: Model set explicitly to "gpt-5"  
  - Inputs: Linked as AI Language Model for Ticket Summarizer node  
  - Credentials: OpenAI API key configured  
  - Failure Modes: API limits, invalid credentials, or model deprecation  

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Validates and parses AI-generated JSON output according to schema for consistent downstream processing.  
  - Configuration: JSON schema example provided with keys "Ticket Summary" and "Proposed Solution"  
  - Inputs: Connected to Ticket Summarizer AI output parser channel  
  - Failure Modes: Parsing errors if AI response deviates from schema  

---

#### 2.5 Aggregation & Formatting

**Overview:**  
Aggregates all individual ticket summaries into one dataset and formats the combined summaries into a human-readable email body with direct Jira ticket links.

**Nodes Involved:**  
- Set Output  
- Aggregate  
- Format Body  

**Node Details:**

- **Set Output**  
  - Type: Set Node  
  - Role: Structures the AI summary output into a uniform JSON format including ticket key, title, description, summary, and proposed solution for each ticket.  
  - Configuration:  
    - JSON output mode with expressions extracting and stringifying fields from AI output and original ticket data.  
  - Inputs: From Merge5 node (merged AI summary and ticket data)  
  - Outputs: To Loop Tickets (back to batch processing)  
  - Failure Modes: Expression errors if fields missing or malformed.  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Collects all processed ticket JSON objects from the batch loop into a single array for final report generation.  
  - Configuration: Aggregates on `output` field containing all ticket summaries.  
  - Inputs: From Loop Tickets (first output path)  
  - Outputs: To Format Body node  
  - Failure Modes: Empty input if no tickets processed; memory constraints for large datasets.  

- **Format Body**  
  - Type: Code Node (JavaScript)  
  - Role: Converts aggregated ticket summaries into a formatted text string listing each ticket with key details, summary, solution, and direct Jira URL.  
  - Configuration:  
    - JS code iterates over aggregated tickets to build multi-line string with labeled fields and Jira ticket links.  
  - Inputs: From Aggregate node  
  - Outputs: To Send Ticket Summaries node  
  - Failure Modes: JS runtime errors or empty inputs causing empty email body.  

---

#### 2.6 Notification Delivery

**Overview:**  
Sends the compiled daily ticket summaries report via Gmail to the configured recipient with a dated subject.

**Nodes Involved:**  
- Send Ticket Summaries  

**Node Details:**

- **Send Ticket Summaries**  
  - Type: Gmail Node  
  - Role: Sends the formatted ticket summary email.  
  - Configuration:  
    - Recipient email is set statically (`n8n_test_result_replace_me@yopmail.com`) and should be updated.  
    - Email subject includes the current date in format `"Daily Ticket Summaries – dd MMM yyyy"`.  
    - Email body is plain text with no attribution appended.  
  - Inputs: From Format Body  
  - Credentials: Gmail OAuth2 Credential configured with appropriate account.  
  - Failure Modes:  
    - Authentication or permission errors in Gmail OAuth2  
    - Invalid recipient email or SMTP errors  
    - Network connectivity issues  

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                        | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                                            |
|------------------------|----------------------------------|-------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start of workflow             | None                        | Set Project Key                 |                                                                                                                                        |
| Schedule Trigger       | Schedule Trigger                 | Scheduled start of workflow          | None                        | Set Project Key                 |                                                                                                                                        |
| Set Project Key        | Set                             | Sets Jira project key "SUP"          | When clicking ‘Test workflow’, Schedule Trigger | Get All Tickets                 |                                                                                                                                        |
| Get All Tickets        | Jira                            | Fetches today's tickets from project | Set Project Key             | Split Out                      | Configuration: Update the Jira credentials with yours.                                                                                 |
| Split Out              | Split Out                       | Splits tickets into individual items | Get All Tickets             | Loop Tickets                   |                                                                                                                                        |
| Loop Tickets           | Split In Batches                | Processes tickets one-by-one          | Split Out, Set Output       | Aggregate, Get Comments from Ticket |                                                                                                                                       |
| Get Comments from Ticket | Jira                            | Retrieves all comments for each ticket | Loop Tickets                | Merge                         |                                                                                                                                        |
| Merge                  | Merge                           | Combines ticket data and comments    | Loop Tickets, Get Comments from Ticket | Ticket Summarizer             |                                                                                                                                        |
| Ticket Summarizer      | LangChain Chain LLM             | AI analysis generating summaries     | Merge                       | Merge5                        |                                                                                                                                        |
| OpenAI Chat Model      | LangChain OpenAI Chat Model     | Provides GPT-5 model for AI          | Linked to Ticket Summarizer | Ticket Summarizer             |                                                                                                                                        |
| Structured Output Parser | LangChain Output Parser Structured | Parses AI JSON output                 | Ticket Summarizer (AI output parser) | Ticket Summarizer             |                                                                                                                                        |
| Merge5                 | Merge                           | Merges AI output with ticket data    | Ticket Summarizer, Merge    | Set Output                    |                                                                                                                                        |
| Set Output             | Set                             | Structures AI output for aggregation | Merge5                      | Loop Tickets                  |                                                                                                                                        |
| Aggregate              | Aggregate                      | Aggregates all ticket summaries      | Loop Tickets                | Format Body                   |                                                                                                                                        |
| Format Body            | Code (JavaScript)               | Formats aggregated summaries for email | Aggregate                   | Send Ticket Summaries         | Gmail - Send Notification: You can adjust the subject & body of the notification here. Also, update the recipient email.               |
| Send Ticket Summaries  | Gmail                          | Sends daily summary email            | Format Body                 | None                         |                                                                                                                                        |
| Sticky Note1           | Sticky Note                    | Describes overall workflow purpose  | None                        | None                         | ## 🎫 Daily Jira Ticket Summarizer using GPT-5 and Jira API - Full feature list and process overview.                                   |
| Sticky Note            | Sticky Note                    | Setup instructions                  | None                        | None                         | ## SETUP REQUIRED - Workflow Configurations and Required Credentials.                                                                   |
| Sticky Note2           | Sticky Note                    | Process overview                    | None                        | None                         | ## 📋 WORKFLOW PROCESS OVERVIEW - Stepwise explanation of the workflow process.                                                         |
| Sticky Note3           | Sticky Note                    | Credential reminder                 | None                        | None                         | Configuration: Update the Jira credentials with yours.                                                                                 |
| Sticky Note4           | Sticky Note                    | Author contact & project info      | None                        | None                         | # 👋 Hi, I’m Billy - Contact info and links for help with n8n and AI automation projects.                                               |
| Sticky Note6           | Sticky Note                    | Gmail send node instructions       | None                        | None                         | ## Gmail - Send Notification - Details for adjusting email subject, body, and recipient.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’` with default settings.
   - Add a **Schedule Trigger** node named `Schedule Trigger`. Configure the interval as desired (e.g., daily at a specific time).

2. **Set Project Key:**
   - Add a **Set** node named `Set Project Key`.
   - Add a string field named `Project Key` with value `SUP`.
   - Connect outputs of both triggers (`When clicking ‘Test workflow’ and Schedule Trigger`) to this node.

3. **Fetch Tickets from Jira:**
   - Add a **Jira** node named `Get All Tickets`.
   - Set operation to `getAll`.
   - Configure the JQL query: `project = {{ $json['Project Key'] }} AND created >= startOfDay()`.
   - Set `Return All` to true.
   - Add Jira Software Cloud API credentials.
   - Connect `Set Project Key` output to this node.

4. **Split Tickets:**
   - Add a **Split Out** node named `Split Out`.
   - Configure to split on field `key`.
   - Include fields: `fields.description, fields.summary`.
   - Connect `Get All Tickets` output to this node.

5. **Process Tickets in Batches:**
   - Add a **Split In Batches** node named `Loop Tickets`.
   - Keep default batch size (1).
   - Connect `Split Out` output to this node.

6. **Retrieve Comments per Ticket:**
   - Add a **Jira** node named `Get Comments from Ticket`.
   - Set operation to `getAll` on resource `issueComment`.
   - Set `Issue Key` to `={{ $json.key }}`.
   - Set `Return All` to true.
   - Use Jira Software Cloud API credentials.
   - Connect `Loop Tickets` output to this node.

7. **Merge Ticket Data with Comments:**
   - Add a **Merge** node named `Merge`.
   - Set mode to `Combine All`.
   - Connect the `Get Comments from Ticket` node as secondary input.
   - Connect the `Loop Tickets` node as primary input.
   - Connect Merge output to AI summarization node next.

8. **Add AI Summarization:**
   - Add a **LangChain Chain LLM** node named `Ticket Summarizer`.
   - Configure prompt with template including ticket title, description, comments, and instruction for JSON output with `Ticket Summary` and `Proposed Solution`.
   - Enable Output Parser with JSON schema for expected fields.
   - Connect `Merge` node output to this node.
   
   - Add a **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.
   - Set the model to `gpt-5`.
   - Connect as AI Language Model input to `Ticket Summarizer`.
   - Add OpenAI API credentials.

   - Add a **LangChain Output Parser Structured** node named `Structured Output Parser`.
   - Provide JSON schema matching expected output.
   - Connect as AI Output Parser input to `Ticket Summarizer`.

9. **Merge AI Output with Ticket Data:**
   - Add a **Merge** node named `Merge5`.
   - Set mode to `Combine All`.
   - Connect primary input to `Ticket Summarizer` output.
   - Connect secondary input to `Merge` node output.
   - Connect output to `Set Output`.

10. **Format AI Output for Aggregation:**
    - Add a **Set** node named `Set Output`.
    - Configure to output JSON with keys: `Ticket Summary`, `Proposed Solution`, `Key`, `Title`, and `Description` extracted from AI output and ticket fields.
    - Connect `Merge5` output to this node.
    - Connect `Set Output` output back into the second input of `Loop Tickets` (to feed aggregation) and to the first output for aggregation.

11. **Aggregate Summarized Tickets:**
    - Add an **Aggregate** node named `Aggregate`.
    - Configure to aggregate on the `output` field collecting all ticket summaries.
    - Connect first output of `Loop Tickets` to this node.

12. **Format Email Body:**
    - Add a **Code** node named `Format Body`.
    - Use JavaScript code to iterate aggregated tickets and create a formatted text body including ticket key, title, description, summary, solution, and a Jira link.
    - Connect `Aggregate` output to this node.

13. **Send Email Notification:**
    - Add a **Gmail** node named `Send Ticket Summaries`.
    - Configure recipient email (update from placeholder to real address).
    - Set subject to: `Daily Ticket Summaries – {{ $now.format('dd MMM yyyy') }}`.
    - Set message body to the formatted text from `Format Body`.
    - Use Gmail OAuth2 credentials.
    - Connect `Format Body` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| ## 🎫 Daily Jira Ticket Summarizer using GPT-5 and Jira API - This template automatically fetches tickets, summarizes with GPT-5, and emails daily reports. | Sticky Note1 in workflow                           |
| ## SETUP REQUIRED - Customize JQL, email recipient, and AI prompts as needed. Required credentials: Jira Software Cloud API, Gmail OAuth2, and OpenAI API. | Sticky Note in workflow                            |
| ## 📋 WORKFLOW PROCESS OVERVIEW - Detailed stepwise description of the workflow steps from trigger to email delivery.                                        | Sticky Note2 in workflow                           |
| Configuration: Update the Jira credentials with your own API credentials to enable Jira API access.                                                        | Sticky Note3                                       |
| ## Gmail - Send Notification - Adjust subject, body, and recipient email address in the send node to suit your needs.                                       | Sticky Note6                                       |
| # 👋 Hi, I’m Billy - Contact info for expert help on n8n and AI automation projects. Email: billychartanto@gmail.com, Website: billychristi.com/n8n       | Sticky Note4                                       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal or protected material. All data processed is lawful and public.

---