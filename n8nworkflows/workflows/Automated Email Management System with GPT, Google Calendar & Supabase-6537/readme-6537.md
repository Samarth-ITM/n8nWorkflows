Automated Email Management System with GPT, Google Calendar & Supabase

https://n8nworkflows.xyz/workflows/automated-email-management-system-with-gpt--google-calendar---supabase-6537


# Automated Email Management System with GPT, Google Calendar & Supabase

---

# Automated Email Management System with GPT, Google Calendar & Supabase

---

### 1. Workflow Overview

This workflow automates email management by integrating Gmail, OpenAI GPT models, Google Calendar, and a Supabase-powered knowledge base to classify emails, draft tailored replies, book calls, and notify via Telegram. It is designed for marketing agencies or businesses that receive diverse inbound emails including sales leads, client communications, reports, and billing inquiries.

**Target Use Cases:**

- Automatically categorize incoming emails into predefined labels (Sales, Clients, Reports, Billing, Other).
- For new sales inquiries or leads, automatically check calendar availability and draft booking replies.
- For informational questions, search a knowledge base and draft informative email replies.
- Notify the user on Telegram about new leads and drafted replies.
- Maintain draft emails in Gmail for review before sending.
- Mark processed emails as read and label accordingly.

**Logical Blocks:**

- **1.1 Input Reception & Email Classification:** Listens to new Gmail messages, classifies email topics, and applies Gmail labels.
- **1.2 Email Thread Context Extraction:** Retrieves full thread data and extracts message bodies for AI processing.
- **1.3 Email Type Classification (Calendar vs. Information):** Uses AI to decide if email requires calendar handling or knowledge-base response.
- **1.4 Calendar Agent Workflow:** Checks Google Calendar availability, drafts booking replies, creates calendar events.
- **1.5 Knowledge Agent Workflow:** Searches the Supabase vector store knowledge base and drafts informative email replies.
- **1.6 Reply Handling & Notifications:** Creates draft replies in Gmail and sends Telegram notifications to alert the user.
- **1.7 Email Post-Processing:** Applies labels, marks emails as read, orchestrates escalation logic for uncertain AI responses.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Email Classification

**Overview:**  
This block triggers the workflow on new Gmail inbox emails, classifies them into high-level categories using an AI text classifier, and applies corresponding Gmail labels.

**Nodes Involved:**  
- Gmail Trigger  
- Text Classifier  
- Sales / Leads (Gmail)  
- Client Comm (Gmail)  
- Reports / Results (Gmail)  
- Billing / Admin (Gmail)  
- Other (Gmail)  
- Mark as Read3 (Gmail)

**Node Details:**

- **Gmail Trigger**  
  - *Type:* Gmail Trigger  
  - *Role:* Watches inbox for new emails every minute, excluding spam/trash.  
  - *Config:* Filters for label "INBOX", polling every minute.  
  - *Outputs:* Email metadata and snippet to Text Classifier.  
  - *Potential Failures:* OAuth token expiry, Gmail API rate limits.  

- **Text Classifier**  
  - *Type:* Langchain Text Classifier  
  - *Role:* Classifies email content into categories: Sales/Leads/Inquiries, Clients/Active Projects, Reports/Results, Billing/Admin, Other/Low Priority.  
  - *Config:* Input text combines sender, subject, and snippet. Categories have descriptive labels.  
  - *Outputs:* Classification result determines downstream Gmail label application.  
  - *Potential Failures:* Model API call failure, misclassification edge cases.  

- **Sales / Leads, Client Comm, Reports / Results, Billing / Admin, Other**  
  - *Type:* Gmail node (Add Labels)  
  - *Role:* Adds respective Gmail labels to the email based on classification.  
  - *Config:* Uses Gmail OAuth2 credentials, applies label ids specific to each category, marks message id from trigger.  
  - *Outputs:* Each leads to Mark as Read3 or further processing.  
  - *Potential Failures:* Label not existing, permission errors, token expiry.  

- **Mark as Read3**  
  - *Type:* Gmail node (Mark as Read)  
  - *Role:* Marks email as read to prevent reprocessing.  
  - *Config:* Uses message id from Gmail Trigger.  
  - *Outputs:* No further downstream connections.  
  - *Potential Failures:* Token or API issues.  

---

#### 1.2 Email Thread Context Extraction

**Overview:**  
Retrieves the full Gmail thread for the incoming email and extracts all message bodies to provide full context to AI models.

**Nodes Involved:**  
- Gmail  
- Get Thread (Code)

**Node Details:**

- **Gmail**  
  - *Type:* Gmail (Thread Get)  
  - *Role:* Fetch complete thread data for the email thread id.  
  - *Config:* Uses threadId from Gmail Trigger, OAuth2 credentials.  
  - *Outputs:* Thread JSON with all messages to Get Thread node.  
  - *Potential Failures:* Thread not found, API rate limits.  

- **Get Thread**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Extracts plain text and HTML bodies from each message part recursively for full context.  
  - *Config:* Custom JavaScript to handle multi-part MIME messages and fallback to snippet if no body found.  
  - *Outputs:* Single JSON item with array of all messages including fullBody, snippet, subject, from, date.  
  - *Potential Failures:* Unexpected MIME structure, missing fields.  

---

#### 1.3 Email Type Classification (Calendar vs. Information)

**Overview:**  
Determines whether the email requires calendar handling (e.g., booking calls) or an informative reply.

**Nodes Involved:**  
- Calendar or Email (Langchain Chain LLM)  
- Structured Output Parser6  
- Switch  

**Node Details:**

- **Calendar or Email**  
  - *Type:* Langchain Chain LLM  
  - *Role:* Uses AI prompt to classify email as "Calendar" (request to book a call) or "Information" (general info request).  
  - *Config:* Prompt includes email snippet and thread messages for context.  
  - *Outputs:* JSON with boolean "calendar" field.  
  - *Potential Failures:* Model API errors, ambiguous emails causing wrong classification.  

- **Structured Output Parser6**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses AI raw response to structured JSON including "calendar" boolean.  
  - *Config:* Manual JSON schema expecting a boolean field.  
  - *Outputs:* Passed to Switch node.  
  - *Potential Failures:* Parsing errors if AI output malformed.  

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Routes flow based on calendar boolean: "Calendar" output leads to Calendar Agent, "Email" output leads to Knowledge Agent.  
  - *Config:* Condition checks $json.output.calendar true/false.  
  - *Outputs:* Two branches, one for calendar emails, one for information emails.  
  - *Potential Failures:* Missing or invalid JSON input causing routing errors.  

---

#### 1.4 Calendar Agent Workflow

**Overview:**  
Handles emails related to booking calls by checking Google Calendar availability, drafting replies, optionally creating calendar events, and notifying the user.

**Nodes Involved:**  
- Calendar Agent (Langchain Agent)  
- get_events (Google Calendar Tool)  
- think_tool (Langchain Think Tool)  
- OpenAI Chat Model5  
- Structured Output Parser  
- Gmail2 (Draft Reply)  
- Inbox Bot (Telegram)  

**Node Details:**

- **Calendar Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Processes email snippet and thread, uses system prompt to guide AI in checking calendar, booking calls, and drafting replies.  
  - *Config:* System message includes detailed rules for handling calendar queries, time zone, polite reply style, and calendar link.  
  - *Tools Used:* Calls get_events to fetch availability, think_tool for internal reasoning.  
  - *Outputs:* Draft reply content and calendar event data.  
  - *Potential Failures:* API call failure, misunderstanding date/time in email, calendar API failures.

- **get_events**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Fetches calendar events within a date range around requested date.  
  - *Config:* Uses AI override expressions for timeMin/timeMax based on date requested, Google Calendar OAuth2 credentials.  
  - *Outputs:* Events data fed back into Calendar Agent.  
  - *Potential Failures:* Date parsing errors, Google API limits or auth issues.  

- **think_tool**  
  - *Type:* Langchain Think Tool  
  - *Role:* Allows Calendar Agent to perform logical reasoning steps internally.  
  - *Config:* No parameters; invoked by Calendar Agent.  
  - *Outputs:* Feeding thoughts back into Agent.  

- **OpenAI Chat Model5**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Language model used by Calendar Agent to generate draft emails and reasoning.  
  - *Config:* Model: gpt-4.1; OpenAI API credentials required.  
  - *Outputs:* Text output to Structured Output Parser.  

- **Structured Output Parser**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses calendar agent output into structured JSON fields: emailBody (HTML), eventName, startTime, endTime, bookCall (boolean), calendarAvailabilities.  
  - *Config:* Manual JSON schema defined for email and event info.  
  - *Outputs:* Parsed data used to draft email and create calendar event.  
  - *Potential Failures:* Parsing failure if AI output malformed.  

- **Gmail2**  
  - *Type:* Gmail (Draft email)  
  - *Role:* Creates a draft reply in Gmail with generated emailBody, linked to original thread, sets reply-to sender.  
  - *Config:* Uses draft resource, HTML email type, OAuth2 credentials.  
  - *Outputs:* Triggers Telegram notification.  
  - *Potential Failures:* Gmail API limits, draft creation failure.  

- **Inbox Bot (Telegram)**  
  - *Type:* Telegram node  
  - *Role:* Sends Telegram notification about new lead with calendar reply drafted.  
  - *Config:* Chat ID configured, message text includes sender name and summary.  
  - *Outputs:* No further nodes.  
  - *Potential Failures:* Telegram API or connectivity issues.  

---

#### 1.5 Knowledge Agent Workflow

**Overview:**  
Handles informational email replies by searching a vectorized knowledge base (Supabase), drafting a detailed reply, and escalating complex queries.

**Nodes Involved:**  
- Knowledge Agent (Langchain Agent)  
- OpenAI Chat Model1  
- Supabase Vector Store2  
- Embeddings OpenAI2  
- Knowledge Database (Langchain Tool Vector Store)  
- OpenAI Chat Model2  
- Structured Output Parser2  
- Escalate? (If node)  
- Gmail3 (Draft Reply)  
- Telegram2 (Notification)

**Node Details:**

- **Knowledge Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Uses email snippet and full thread, queries knowledge database, drafts reply with answers or escalates.  
  - *Config:* System prompt instructs not to hallucinate, to set escalate true if unsure, to sign off as Abdul; date/time context in Chicago.  
  - *Outputs:* Draft reply content and escalation flag.  
  - *Potential Failures:* Model failures, vector store query errors.  

- **OpenAI Chat Model1**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Used to query knowledge database and generate replies.  
  - *Config:* Model: gpt-4.1-mini, OpenAI API credentials.  
  - *Outputs:* Embeddings for vector store query.  

- **Supabase Vector Store2**  
  - *Type:* Langchain Vector Store (Supabase)  
  - *Role:* Stores and retrieves relevant documents for knowledge-based replies.  
  - *Config:* Table name "documents", query "match_documents", Supabase API credentials.  
  - *Outputs:* Matches fed to Knowledge Database tool.  

- **Embeddings OpenAI2**  
  - *Type:* Langchain Embeddings OpenAI  
  - *Role:* Generates embeddings for queries to vector store.  
  - *Config:* OpenAI API credentials.  

- **Knowledge Database**  
  - *Type:* Langchain Tool Vector Store  
  - *Role:* RAG tool to find answers in knowledge base for composing replies.  
  - *Config:* TopK=5 results, description with FAQ and services content.  
  - *Outputs:* Used by Knowledge Agent for response.  

- **OpenAI Chat Model2**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Generates final email reply from knowledge base data.  
  - *Config:* Model: gpt-4.1-mini, OpenAI API credentials.  

- **Structured Output Parser2**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses output to fields: emailBody (HTML), reason, escalate (boolean), knowledgeDatabase summary.  
  - *Config:* Manual JSON schema with required fields.  
  - *Outputs:* Passed to escalation logic.  

- **Escalate?**  
  - *Type:* If node  
  - *Role:* Checks if escalate flag true. If yes, sends Telegram2 notification. If no, proceeds to Gmail3 draft node.  
  - *Config:* Condition on parsed escalate boolean.  
  - *Outputs:* Two branches for escalation or normal reply.  

- **Gmail3**  
  - *Type:* Gmail (Draft email)  
  - *Role:* Creates draft reply email for knowledge responses.  
  - *Config:* HTML email, reply-to sender, thread ID, OpenAI OAuth2 credentials.  

- **Telegram2**  
  - *Type:* Telegram node  
  - *Role:* Notifies user that unsure reply drafted requiring manual review.  
  - *Config:* Chat ID, customized message.  

---

#### 1.6 Reply Handling & Notifications

**Overview:**  
Sends notifications to Telegram about new leads, differentiating between calendar booking leads and knowledge-base inquiries.

**Nodes Involved:**  
- Inbox Bot (Telegram)  
- Telegram2  

**Node Details:**

- Both Telegram nodes send formatted notifications with lead name and status.  
- Messages encourage user to check Gmail drafts.  
- Configured with Telegram bot API credentials and chat ID.  
- Potential failures include API connectivity issues.

---

#### 1.7 Email Post-Processing

**Overview:**  
Handles final labeling and marking emails as read to maintain inbox hygiene.

**Nodes Involved:**  
- Sales / Leads, Client Comm, Reports / Results, Billing / Admin, Other (Gmail label nodes)  
- Mark as Read3  

**Node Details:**

- Labels emails based on classification outcome using Gmail API.  
- Marks emails as read after labeling.  
- Ensures emails are not repeatedly processed.  

---

### 3. Summary Table

| Node Name           | Node Type                                | Functional Role                           | Input Node(s)          | Output Node(s)                  | Sticky Note                                                                                   |
|---------------------|-----------------------------------------|-----------------------------------------|------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| Gmail Trigger       | Gmail Trigger                           | Watches inbox for new emails             | -                      | Text Classifier                |                                                                                               |
| Text Classifier     | Langchain Text Classifier               | Classifies emails into categories        | Gmail Trigger          | Sales / Leads, Client Comm, Reports / Results, Billing / Admin, Other |                                                                                               |
| Sales / Leads       | Gmail (Add Labels)                      | Labels sales/inquiry emails               | Text Classifier        | Gmail                        |                                                                                               |
| Client Comm         | Gmail (Add Labels)                      | Labels client communication emails       | Text Classifier        | Gmail                        |                                                                                               |
| Reports / Results   | Gmail (Add Labels)                      | Labels report/result emails               | Text Classifier        | Gmail                        |                                                                                               |
| Billing / Admin     | Gmail (Add Labels)                      | Labels billing/admin emails               | Text Classifier        | Gmail                        |                                                                                               |
| Other               | Gmail (Add Labels)                      | Labels other/low priority emails          | Text Classifier        | Mark as Read3                |                                                                                               |
| Mark as Read3       | Gmail (Mark as Read)                    | Marks processed email as read             | Other                  | -                             |                                                                                               |
| Gmail               | Gmail (Thread Get)                      | Retrieves full email thread               | Sales / Leads          | Get Thread                   |                                                                                               |
| Get Thread          | Code                                   | Extracts full message bodies              | Gmail                   | Calendar or Email            |                                                                                               |
| Calendar or Email   | Langchain Chain LLM                    | Classifies email as calendar or info     | Get Thread              | Switch                      |                                                                                               |
| Structured Output Parser6 | Langchain Output Parser Structured | Parses calendar boolean classification    | Calendar or Email       | Switch                      |                                                                                               |
| Switch              | Switch                                 | Routes flow to Calendar or Knowledge agent | Structured Output Parser6 | Calendar Agent, Knowledge Agent |                                                                                               |
| Calendar Agent      | Langchain Agent                        | Handles calendar bookings and replies    | Switch                  | Gmail2                      | Sticky Note1: "# Check Calendar Availability + Draft Email"                                   |
| get_events          | Google Calendar Tool                   | Gets calendar availability                | Calendar Agent (ai_tool) | Calendar Agent              |                                                                                               |
| think_tool          | Langchain Think Tool                   | Internal reasoning for calendar agent    | Calendar Agent (ai_tool) | Calendar Agent              |                                                                                               |
| OpenAI Chat Model5  | Langchain OpenAI Chat Model            | Language model for calendar agent         | Calendar Agent          | Structured Output Parser     |                                                                                               |
| Structured Output Parser | Langchain Output Parser Structured  | Parses calendar agent output               | OpenAI Chat Model5      | Gmail2                      |                                                                                               |
| Gmail2              | Gmail (Draft email)                    | Creates draft reply email for calendar    | Structured Output Parser | Inbox Bot                  |                                                                                               |
| Inbox Bot           | Telegram                              | Sends Telegram notification for calendar  | Gmail2                   | -                           |                                                                                               |
| Knowledge Agent     | Langchain Agent                      | Handles knowledge base replies             | Switch                   | Escalate?                   | Sticky Note2: "# Check Knowledge Base + Draft Email"                                          |
| OpenAI Chat Model1  | Langchain OpenAI Chat Model            | Language model for knowledge DB queries   | Knowledge Agent          | Knowledge Database           |                                                                                               |
| Supabase Vector Store2 | Langchain Vector Store Supabase       | Queries knowledge base for info           | Embeddings OpenAI2       | Knowledge Database           |                                                                                               |
| Embeddings OpenAI2  | Langchain Embeddings OpenAI            | Creates embeddings for vector DB queries  | -                        | Supabase Vector Store2       |                                                                                               |
| Knowledge Database  | Langchain Tool Vector Store            | RAG tool for knowledge base retrieval     | OpenAI Chat Model1       | Knowledge Agent              |                                                                                               |
| OpenAI Chat Model2  | Langchain OpenAI Chat Model            | Generates knowledge-based email replies   | Knowledge Agent          | Structured Output Parser2    |                                                                                               |
| Structured Output Parser2 | Langchain Output Parser Structured | Parses knowledge agent output              | OpenAI Chat Model2       | Escalate?                   |                                                                                               |
| Escalate?           | If                                     | Checks if escalation needed                | Structured Output Parser2 | Telegram2, Gmail3           |                                                                                               |
| Gmail3              | Gmail (Draft email)                    | Drafts reply email for knowledge agent    | Escalate? (false branch) | Telegram                    |                                                                                               |
| Telegram2           | Telegram                              | Sends Telegram notification on escalation | Escalate? (true branch)  | -                           |                                                                                               |
| Structured Output Parser | Langchain Output Parser Structured  | Parses calendar agent output                | OpenAI Chat Model5      | Gmail2                      |                                                                                               |
| OpenAI Chat Model7  | Langchain OpenAI Chat Model            | Used by Calendar or Email node             | Get Thread               | Calendar or Email           |                                                                                               |
| Window Buffer Memory | Langchain Memory Buffer Window         | Maintains session conversation context    | -                        | -                           |                                                                                               |
| Sticky Note1        | Sticky Note                           | "# Check Calendar Availability + Draft Email" | -                       | -                           |                                                                                               |
| Sticky Note2        | Sticky Note                           | "# Check Knowledge Base + Draft Email"    | -                        | -                           |                                                                                               |
| Sticky Note3        | Sticky Note                           | "Example Customer Email (image)"           | -                        | -                           |                                                                                               |
| Sticky Note4        | Sticky Note                           | "# Categorize incoming emails"             | -                        | -                           |                                                                                               |
| Sticky Note5        | Sticky Note                           | "# New lead? Send to calendar/knowledge agent" | -                      | -                           |                                                                                               |
| Sticky Note6        | Sticky Note                           | "Hey, I'm Abdul 👋 ... https://www.builtbyabdul.com/" | -                       | -                           |                                                                                               |
| Sticky Note7        | Sticky Note                           | "# AI Inbox Manager ... How to customize ..." | -                      | -                           |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure to watch inbox label "INBOX", exclude spam/trash  
   - Poll every minute  
   - Connect Gmail OAuth2 credentials  

2. **Add Text Classifier Node**  
   - Type: Langchain Text Classifier  
   - Input: Combine fields `From`, `Subject`, and `snippet` from Gmail Trigger  
   - Define categories: Sales/Leads/Inquiries, Clients/Active Projects, Reports/Results, Billing/Admin, Other/Low Priority with descriptions  
   - Connect OpenAI (or supported) API credentials  

3. **Add Gmail Label Nodes for Each Category**  
   - For each category, create a Gmail node to add corresponding label by label ID  
   - Use messageId from Gmail Trigger  
   - Connect Gmail OAuth2 credentials  
   - Connect outputs from Text Classifier to these nodes  

4. **Add Mark as Read Node**  
   - Gmail node to mark email as read  
   - Use message ID from Gmail Trigger  
   - Connect output from "Other" label node (or last label node in chain)  

5. **Add Gmail Thread Get Node**  
   - Gmail node to get full thread by threadId from Gmail Trigger  
   - Include OAuth2 credentials  

6. **Add Code Node to Extract Full Thread Messages**  
   - Paste JavaScript code to parse multi-part email messages and extract body text  
   - Input from Gmail Thread Get node  

7. **Add 'Calendar or Email' Langchain Chain LLM Node**  
   - Prompt to classify email as Calendar (booking) or Information (general)  
   - Include entire thread messages and snippet as input  
   - Use OpenAI API credentials  

8. **Add Structured Output Parser Node**  
   - Define JSON schema expecting boolean "calendar" field  
   - Connect output of Chain LLM  

9. **Add Switch Node**  
   - Two outputs: Calendar (true), Email (false) based on parsed "calendar" field  
   - Connect from Structured Output Parser  

10. **Calendar Agent Branch:**  
    - Add Langchain Agent Node "Calendar Agent"  
    - System prompt includes rules about booking calls, time zones, calendar link, and tone  
    - Connect to tools: Google Calendar Tool (get_events), Think Tool  
    - Add Google Calendar Tool Node "get_events" configured to query availability based on AI-specified date range  
    - Connect Google Calendar OAuth2 credentials  
    - Add Think Tool node for internal reasoning  
    - Add OpenAI Chat Model node for agent LLM (model: gpt-4.1)  
    - Add Structured Output Parser for calendar agent output with relevant JSON schema (emailBody, eventName, startTime, endTime, bookCall)  
    - Add Gmail Draft node to create email draft reply in original thread, reply-to sender address  
    - Add Telegram node to notify user of new lead and drafted calendar reply  

11. **Knowledge Agent Branch:**  
    - Add Langchain Agent Node "Knowledge Agent"  
    - System prompt instructs to query knowledge base, draft reply, escalate if unsure  
    - Add OpenAI Chat Model node for embeddings generation  
    - Add Supabase Vector Store node configured with "documents" table and Supabase credentials  
    - Add Langchain Tool Vector Store node "Knowledge Database" with topK=5 and description of FAQ/services  
    - Add OpenAI Chat Model node for generating replies from knowledge base (model: gpt-4.1-mini)  
    - Add Structured Output Parser node with schema for emailBody, reason, escalate boolean  
    - Add If node "Escalate?" to branch on escalate true/false  
    - On false branch, add Gmail Draft node for reply email  
    - On true branch, add Telegram node to notify user of escalation needed  

12. **Add Sticky Notes for Documentation**  
    - For categorization  
    - For calendar agent workflow  
    - For knowledge agent workflow  
    - For example customer email and draft screenshots  

13. **Set up Credentials:**  
    - Gmail OAuth2 with appropriate scopes for reading threads, adding labels, drafting emails  
    - OpenAI API key for all AI nodes  
    - Google Calendar OAuth2 for calendar access  
    - Supabase API key for vector database  
    - Telegram Bot API key and chat ID for notifications  

14. **Test Workflow End-to-End:**  
    - Send test emails for each category  
    - Verify labels applied, drafts created, calendar events booked, Telegram notifications sent  
    - Confirm email reply drafts contain AI-generated content fitting the email category and context  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| AI Inbox Manager that categorizes emails, books calls, and replies automatically. Designed for marketing agencies but adaptable broadly.     |                                                                 |
| Workflow uses RAG approach with Supabase vector store to provide factual, non-hallucinated replies.                                          |                                                                 |
| Telegram notifications keep user informed about new leads and escalations without checking email constantly.                                |                                                                 |
| Example draft email and Telegram notification screenshots available at:                                                                       | https://i.imgur.com/ghJlqjU.png and https://i.imgur.com/nbIMIAZ.png |
| Example customer email screenshot:                                                                                                          | https://i.imgur.com/JaS0BsG.png                      |
| Developer contact and branding: Abdul, website https://www.builtbyabdul.com                                                                  |                                                                 |
| Requires n8n nodes: Gmail, Langchain (Text Classifier, Agent, Chain LLM, Output Parser, Think Tool, Memory Buffer), Google Calendar Tool, Telegram, Supabase Vector Store, OpenAI embeddings and chat models |                                                                 |

---

**Disclaimer:** The provided content is derived solely from an automated workflow built with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected material. All data processed is legal and public.

---