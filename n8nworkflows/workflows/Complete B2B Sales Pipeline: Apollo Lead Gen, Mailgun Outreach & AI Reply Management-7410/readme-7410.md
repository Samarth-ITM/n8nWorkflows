Complete B2B Sales Pipeline: Apollo Lead Gen, Mailgun Outreach & AI Reply Management

https://n8nworkflows.xyz/workflows/complete-b2b-sales-pipeline--apollo-lead-gen--mailgun-outreach---ai-reply-management-7410


# Complete B2B Sales Pipeline: Apollo Lead Gen, Mailgun Outreach & AI Reply Management

### 1. Workflow Overview

This workflow implements a **Complete B2B Sales Pipeline** integrating lead generation, outreach, and AI-driven reply management. It orchestrates Apollo.io lead scraping, Mailgun email outreach, and AI-based intelligent response handling using OpenAI and Anthropic language models. The pipeline is designed for automating sales prospecting and follow-up at scale, with scheduling, batching, and conditional processing blocks to manage volume and timing.

Logical blocks:

- **1.1 Lead Generation and Filtering:** Scrapes and processes leads from Apollo and other sources, filters verified and new leads, and stores them in Supabase.
- **1.2 Outreach Preparation & Sending:** Generates personalized email sequences using AI agents and sends emails via Mailgun in controlled batches with wait nodes for pacing.
- **1.3 AI Reply Management:** Listens for incoming replies via Gmail triggers, parses responses, generates professional draft replies with AI, and updates reply status in a Postgres database.
- **1.4 Scheduling and Control:** Uses schedule triggers and limit/switch nodes to control batch sizes, timing, and manage parallel processing paths.
- **1.5 Supporting Utilities:** Includes timezone handling, code nodes for processing logic, and Telegram notifications for monitoring.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Generation and Filtering

- **Overview:** Retrieves leads data, filters out already scraped and non-verified emails, stores new leads in Supabase, and prepares them for outreach.
- **Nodes Involved:** Schedule Trigger1, Supabase2, Sort2, If2, Limit1, Switch, Loop Over Items, Supabase3, Supabase4, Run an Actor, Extract Info, Only Keep Verified Emails, Keep only the new leads, Create rows with new leads, Set Telegram message, Confirmation message
- **Node Details:**

  - **Schedule Trigger1**  
    - Type: scheduleTrigger  
    - Role: Periodically initiates lead retrieval from the database.  
    - Config: Runs at configured intervals (default cron or fixed delay).  
    - Failure modes: Scheduler failure or misconfiguration leads to no lead update.

  - **Supabase2**  
    - Type: supabase  
    - Role: Retrieves lead records from Supabase database.  
    - Config: Query set to fetch raw leads with required columns, e.g., email, company info.  
    - Potential issues: Auth errors, query timeouts.

  - **Sort2**  
    - Type: sort  
    - Role: Sorts lead records for deterministic processing.  
    - Config: Sort by relevant lead attribute (e.g., date added).  
    - Edge cases: Empty input array.

  - **If2**  
    - Type: if  
    - Role: Conditional processing to check if leads exist or meet criteria.  
    - Config: Checks for non-empty lead list or specific flags.  
    - Failure: Expression errors, empty inputs.

  - **Limit1**  
    - Type: limit  
    - Role: Limits the number of leads processed per execution to control throughput.  
    - Config: Set to a max number (e.g., 100).  
    - Edge cases: Limit too low or too high impacting throughput.

  - **Switch**  
    - Type: switch  
    - Role: Routes leads to different processing paths based on criteria (e.g., lead type).  
    - Config: Conditions based on lead properties.

  - **Loop Over Items / Loop Over Items1**  
    - Type: splitInBatches  
    - Role: Processes leads in manageable batches.  
    - Config: Batch size configured (e.g., 10-20).  
    - Failures: Large batches cause timeouts.

  - **Supabase3, Supabase4**  
    - Type: supabase  
    - Role: Writes or updates lead information in Supabase after filtering.  
    - Config: Insert or update queries.  
    - Failures: DB write failures, constraint violations.

  - **Run an Actor**  
    - Type: apify  
    - Role: Executes an external Apify actor for lead enrichment or scraping.  
    - Config: Actor ID, input parameters dynamically generated.  
    - Failures: Actor run failure, API limits.

  - **Extract Info**  
    - Type: set  
    - Role: Extracts relevant fields from Apify output for further processing.  
    - Config: Field mappings.

  - **Only Keep Verified Emails**  
    - Type: filter  
    - Role: Filters leads to keep only those with verified email addresses.  
    - Config: Filter condition on email verification flag.

  - **Keep only the new leads**  
    - Type: compareDatasets  
    - Role: Compares new leads with existing leads to keep only new ones.  
    - Config: Key fields for comparison (email, company).  
    - Edge cases: Data mismatch or duplicates.

  - **Create rows with new leads**  
    - Type: supabase  
    - Role: Inserts new lead rows into Supabase for outreach.  
    - Config: Insert query with new leads.

  - **Set Telegram message + Confirmation message**  
    - Type: set + telegram  
    - Role: Sends Telegram notifications confirming how many new leads were added.  
    - Config: Message template with dynamic data.

---

#### 1.2 Outreach Preparation & Sending

- **Overview:** Generates AI-crafted personalized email sequences for leads, sends emails via Mailgun in controlled batches, and queues follow-ups with timed waits.
- **Nodes Involved:** General anlysis, create email sequence, Code, OpenAI Chat Model, Loop Over Items, Mailgun, Wait, Switch, Limit, Sort, Supabase12, Supabase14, Supabase15, Supabase16, Loop Over Items7, Loop Over Items8, Loop Over Items10, Loop Over Items11, Mailgun6, Mailgun7, Mailgun8, Mailgun9, Wait1, Wait7, Wait8, Wait10, Wait11
- **Node Details:**

  - **General anlysis**  
    - Type: openAi  
    - Role: Performs AI analysis of lead/company data to inform email content.  
    - Config: OpenAI model with prompt tuned for company research.

  - **create email sequence**  
    - Type: langchain agent  
    - Role: Generates a sequence of emails (initial outreach + follow-ups) using AI.  
    - Config: Agent configured with email templates and sales context.

  - **Code**  
    - Type: code  
    - Role: Custom JS to format or manipulate AI output for Mailgun compatibility.

  - **OpenAI Chat Model**  
    - Type: lmChatOpenAi  
    - Role: Core language model for email content generation.

  - **Loop Over Items & variants**  
    - Type: splitInBatches  
    - Role: Batch processing of emails to send with throttling.

  - **Mailgun, Mailgun6, Mailgun7, Mailgun8, Mailgun9**  
    - Type: mailgun  
    - Role: Sends actual emails through Mailgun SMTP API.  
    - Config: Mailgun credentials, sender address set dynamically.

  - **Wait, Wait1, Wait7, Wait8, Wait10, Wait11**  
    - Type: wait  
    - Role: Introduces delays between batches to avoid spam flags or rate limits.

  - **Switch, Limit, Sort**  
    - Type: switch, limit, sort  
    - Role: Controls flow paths, batch sizes, and ordering of emails for sending.

  - **Supabase12, Supabase14, Supabase15, Supabase16**  
    - Type: supabase  
    - Role: Updates lead outreach status and email sent timestamps in database.

---

#### 1.3 AI Reply Management

- **Overview:** Detects inbound email replies via Gmail trigger, parses reply content, uses AI to generate professional draft responses, and updates reply tracking in Postgres.
- **Nodes Involved:** Gmail Trigger, Get a message, Professional Email Response Agent, Email Response Parser, Create Draft Response, Get Email History, Check Sent History, Set replied = Yes, Execute a SQL query, Only from email campaigns, extract email
- **Node Details:**

  - **Gmail Trigger**  
    - Type: gmailTrigger  
    - Role: Watches Gmail inbox for new incoming emails (replies).  
    - Config: OAuth2 credentials, filter for specific labels or inbox.

  - **Get a message**  
    - Type: gmail  
    - Role: Retrieves full email content for processing.

  - **Professional Email Response Agent**  
    - Type: langchain agent  
    - Role: AI agent that composes professional replies based on incoming email context.

  - **Email Response Parser**  
    - Type: outputParserStructured  
    - Role: Parses structured AI output to extract reply components.

  - **Create Draft Response**  
    - Type: gmail  
    - Role: Creates draft reply emails in Gmail for user review or automated sending.

  - **Get Email History, Check Sent History**  
    - Type: gmailTool  
    - Role: Retrieves email thread history and checks whether previous outreach emails were sent to avoid duplicate replies.

  - **Set replied = Yes**  
    - Type: postgres  
    - Role: Marks lead as replied in Postgres DB.

  - **Execute a SQL query, Only from email campaigns, extract email**  
    - Type: postgres, filter, code  
    - Role: Extracts and filters emails relevant to campaigns for reply processing.

  - **Failure modes:**  
    - Gmail API rate limits, OAuth expiration, AI generation timeouts, parsing errors.

---

#### 1.4 Scheduling and Control

- **Overview:** Manages timing, batch sizes, and conditional branching for scalable, rate-limited pipeline execution.
- **Nodes Involved:** Schedule Trigger1, Schedule Trigger3, Schedule Trigger4, Schedule Trigger6, Limit, Limit1, Limit2, Limit3, Limit4, Switch, Switch5, Switch6, Sort, Sort1, Sort2, Sort3, If, If1, If2, If3, No Operation nodes
- **Node Details:**

  - **Schedule Trigger nodes**  
    - Initiate periodic batch processes for different pipeline stages.

  - **Limit nodes**  
    - Enforce maximum processing counts per batch for load management.

  - **Switch nodes**  
    - Route data flows based on lead attributes or campaign stages.

  - **Sort nodes**  
    - Ensure deterministic order of processing, e.g., by date.

  - **If nodes**  
    - Gate execution paths based on data presence or flags.

  - **No Operation nodes**  
    - Used as placeholders or to gracefully handle empty branches.

---

#### 1.5 Supporting Utilities

- **Overview:** Include timezone handling, code for custom logic, Telegram notifications, and AI-powered scraping.
- **Nodes Involved:** Time Zone, Sender Email, Code nodes, Telegram, Langchain memory buffer, Apify actor, Sticky Notes
- **Node Details:**

  - **Time Zone**  
    - Type: code  
    - Role: Adjusts timestamps or scheduling according to timezone.

  - **Sender Email**  
    - Type: code  
    - Role: Dynamically sets sender email address for Mailgun.

  - **Telegram nodes**  
    - Notify user or admin about key events (lead added, confirmation).

  - **Simple Memory (Langchain memory buffer)**  
    - Keeps conversational context for AI agents.

  - **Run an Actor (Apify)**  
    - Runs external scraping or enrichment actor with parameters from code nodes.

  - **Sticky Notes**  
    - Documentation and reminders inside the workflow editor (no operational role).

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                          | Input Node(s)                      | Output Node(s)                     | Sticky Note                           |
|-------------------------------|--------------------------------|----------------------------------------|----------------------------------|----------------------------------|-------------------------------------|
| Limit                         | limit                         | Limits item count for batch processing | Sort1                            | Merge                            |                                     |
| Merge                         | merge                         | Merges multiple data streams           | Limit, Code                      | Time Zone                       |                                     |
| Time Zone                     | code                          | Adjust timezone for timestamps          | Merge                           | Sender Email                    |                                     |
| Sender Email                  | code                          | Sets sender email dynamically           | Time Zone                      | Your leads table                |                                     |
| Switch                       | switch                        | Routes leads by type or condition       | Limit1                         | Loop Over Items, Loop Over Items1|                                     |
| Loop Over Items               | splitInBatches                | Processes leads in batches               | Switch                        | Supabase3, Mailgun1             |                                     |
| Wait                         | wait                          | Waits between batch sends                 | Mailgun1                      | Loop Over Items                 |                                     |
| Loop Over Items1              | splitInBatches                | Processes leads in batches               | Switch                        | Supabase4, Mailgun              |                                     |
| Wait1                        | wait                          | Waits between batch sends                 | Mailgun                       | Loop Over Items1                |                                     |
| Schedule Trigger1             | scheduleTrigger               | Triggers lead retrieval periodically     | -                              | Supabase2                      |                                     |
| If3                          | if                            | Conditional check on leads                 | Get many rows                  | Sort1, No Operation             |                                     |
| Mailgun                      | mailgun                       | Sends emails                             | Loop Over Items1               | Wait1                         |                                     |
| Mailgun1                     | mailgun                       | Sends emails                             | Loop Over Items                | Wait                          |                                     |
| General anlysis              | openAi                        | Analyzes lead/company data for emails    | research about company         | create email sequence          |                                     |
| Limit1                       | limit                         | Limits items per batch                     | If2                           | Switch                        |                                     |
| OpenAI Chat Model            | lmChatOpenAi                  | Generates AI content                       | General anlysis               | create email sequence          |                                     |
| Sort                        | sort                          | Sorts data                                | If1                           | Limit2                        |                                     |
| Sort1                       | sort                          | Sorts data                                | If3                           | Limit                         |                                     |
| No Operation, do nothing     | noOp                          | Placeholder for empty paths               | If3, If2, If1                 | -                            |                                     |
| Supabase2                   | supabase                      | Retrieves lead data                        | Schedule Trigger1             | Sort2                        |                                     |
| If2                         | if                            | Checks lead conditions                     | Sort2                        | Limit1, No Operation          |                                     |
| Supabase3                   | supabase                      | Stores processed leads                     | Loop Over Items               | -                            |                                     |
| Supabase4                   | supabase                      | Stores processed leads                     | Loop Over Items1              | -                            |                                     |
| Run an Actor                | apify                         | Runs external scraping/enrichment actor   | Create URL                   | Extract Info                  |                                     |
| Extract Info                | set                           | Extracts info from actor output            | Run an Actor                 | Only Keep Verified Emails     |                                     |
| Only Keep Verified Emails   | filter                        | Filters leads to verified emails           | Extract Info                 | Keep only the new leads       |                                     |
| Keep only the new leads     | compareDatasets               | Removes duplicates                         | Select already scraped mails  | Already scraped, Create rows  |                                     |
| Create rows with new leads  | supabase                      | Inserts new leads into database            | Keep only the new leads       | Set Telegram message          |                                     |
| Set Telegram message        | set                           | Prepares Telegram notification             | Create rows with new leads    | Confirmation message          |                                     |
| Confirmation message        | telegram                      | Sends Telegram notification                 | Set Telegram message          | -                            |                                     |
| Schedule Trigger3           | scheduleTrigger               | Triggers lead data refresh periodically    | -                           | Get many rows                |                                     |
| Get many rows               | supabase                      | Retrieves many rows from database          | Schedule Trigger3            | If3                         |                                     |
| Gmail Trigger              | gmailTrigger                  | Listens for incoming emails                 | -                           | extract email                |                                     |
| extract email              | code                          | Extracts email from incoming message         | Gmail Trigger               | Execute a SQL query          |                                     |
| Execute a SQL query        | postgres                      | Queries campaign emails                      | extract email               | Only from email campaigns    |                                     |
| Only from email campaigns  | filter                        | Filters emails related to campaigns          | Execute a SQL query          | Set replied = Yes            |                                     |
| Set replied = Yes          | postgres                      | Updates reply status in DB                    | Only from email campaigns    | Get a message                |                                     |
| Get a message              | gmail                         | Retrieves full email content                   | Set replied = Yes            | Professional Email Response Agent |                                     |
| Professional Email Response Agent | langchain agent          | Generates professional email replies          | Get a message               | Create Draft Response        |                                     |
| Email Response Parser      | outputParserStructured        | Parses structured AI reply output              | Professional Email Response Agent | Professional Email Response Agent |                                     |
| Create Draft Response      | gmail                         | Creates draft reply email                       | Professional Email Response Agent | -                          |                                     |
| Get Email History          | gmailTool                     | Retrieves email thread history                 | -                           | Professional Email Response Agent |                                     |
| Check Sent History         | gmailTool                     | Checks sent email history                       | -                           | Professional Email Response Agent |                                     |
| Anthropic Chat Model1      | lmChatAnthropic               | Alternative AI language model                   | -                           | Professional Email Response Agent |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a schedule trigger node** named "Schedule Trigger1" to periodically fetch leads.

2. **Add a Supabase node** "Supabase2" connected to "Schedule Trigger1" to query lead data from your Supabase database (configure credentials and SQL query).

3. **Add a Sort node** "Sort2" connected to "Supabase2" to order leads.

4. **Add an If node** "If2" connected to "Sort2" to check if leads are present or meet criteria.

5. **Add a Limit node** "Limit1" connected to the true branch of "If2" to restrict batch size (e.g., 100).

6. **Add a Switch node** "Switch" connected to "Limit1" to route leads based on type or source.

7. **Add two SplitInBatches nodes** "Loop Over Items" and "Loop Over Items1" connected to "Switch" outputs, setting batch sizes (e.g., 20).

8. **Connect "Loop Over Items" to Supabase nodes "Supabase3" and "Mailgun1"**; "Loop Over Items1" to "Supabase4" and "Mailgun" nodes for data writing and email sending.

9. **Configure Mailgun nodes ("Mailgun", "Mailgun1")** with your Mailgun credentials, sender email dynamically set using a code node ("Sender Email").

10. **Add Wait nodes ("Wait", "Wait1")** connected after each Mailgun node to pace sending batches, configure wait times to avoid spam/rate limits.

11. **Create AI analysis nodes:**  
    - "General anlysis" (OpenAI) for lead/company analysis.  
    - "create email sequence" (Langchain agent) for generating personalized email sequences connected downstream.  
    - "OpenAI Chat Model" to power the generation with proper API credentials.

12. **Add a Code node** to manipulate email sequences if needed.

13. **Add Supabase nodes ("Supabase12", "Supabase14", "Supabase15", "Supabase16")** connected to batch loops to update outreach status and email sent timestamps.

14. **Add scheduling triggers for various pipeline stages** (Schedule Trigger3, Schedule Trigger4, Schedule Trigger6) to refresh data and execute different parts in intervals.

15. **For AI Reply Management:**  
    - Add a "Gmail Trigger" node to listen for incoming emails.  
    - Link to "extract email" code node to parse sender info.  
    - Add "Execute a SQL query" node to check campaign-related emails.  
    - Add filtering node "Only from email campaigns" to narrow down.  
    - Add Postgres node "Set replied = Yes" to mark replied leads.  
    - Connect to "Get a message" Gmail node to fetch full email.  
    - Add "Professional Email Response Agent" (Langchain agent) to generate replies.  
    - Add "Email Response Parser" to parse AI output.  
    - Add "Create Draft Response" Gmail node to draft replies for review.  
    - Optionally add "Get Email History" and "Check Sent History" GmailTool nodes for context.

16. **Add control nodes:**  
    - Limit, Switch, Sort, If nodes to manage flow, batch sizes, and conditional branching.  
    - No Operation nodes to handle empty branches gracefully.

17. **Add utility nodes:**  
    - "Time Zone" code node to handle timezones.  
    - "Sender Email" code node for dynamic sender selection.  
    - Telegram nodes to send notifications on lead additions or confirmations.

18. **Configure credentials:**  
    - Supabase with API keys.  
    - Mailgun with API keys and domain.  
    - OpenAI and Anthropic with API keys.  
    - Gmail OAuth2 credentials with required scopes.  
    - Apify API key for actor execution.

19. **Test each block independently** for data correctness, API connectivity, and error handling.

20. **Optionally add Sticky Notes** throughout workflow for documentation and reminders.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                     | Context or Link                                                                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow integrates Apollo lead gen, Mailgun outreach, and AI reply management for B2B sales automation.                                        | Workflow Title and Description                                                                                                                                               |
| Mailgun nodes are configured with "continueRegularOutput" on error to avoid complete workflow failure on transient email send errors.         | Mailgun nodes configuration                                                                                                                                                  |
| Telegram nodes provide monitoring notifications to alert about new leads and confirmations.                                                     | Telegram integration for monitoring                                                                                                                                          |
| AI agents use Langchain nodes with OpenAI and Anthropic models for flexible email content generation and reply drafting.                       | Langchain AI nodes                                                                                                                                                            |
| Scheduling nodes control batch processing frequency to respect API limits and avoid spam flags.                                                 | Schedule Trigger nodes                                                                                                                                                         |
| Workflow designed to handle large batches with splitInBatches, waits, and limits to ensure scalability and compliance.                         | Batch processing design                                                                                                                                                        |
| External Apify actor is used for lead enrichment/scraping with dynamic input from code nodes.                                                   | Apify integration                                                                                                                                                             |
| Gmail nodes require OAuth2 credentials with full Gmail API access and proper webhook setup for triggers.                                        | Gmail integration setup                                                                                                                                                        |
| Langchain memory buffer node is used to maintain conversational context for scraping or AI agents to improve results.                          | Simple Memory node                                                                                                                                                             |
| Sticky notes scattered in workflow serve as inline documentation for future maintainers.                                                       | Internal workflow documentation (no operational effect)                                                                                                                      |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.