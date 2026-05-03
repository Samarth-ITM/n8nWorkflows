Generate Personalized Music Promotion Emails with GPT-5 and Gmail Context Analysis

https://n8nworkflows.xyz/workflows/generate-personalized-music-promotion-emails-with-gpt-5-and-gmail-context-analysis-8884


# Generate Personalized Music Promotion Emails with GPT-5 and Gmail Context Analysis

### 1. Workflow Overview

This workflow automates the generation of personalized music promotion emails by leveraging AI (GPT-5 models) and Gmail conversation history analysis. It targets music industry professionals who want to efficiently send tailored booking or promotional emails to contacts such as venues, festivals, clubs, media outlets, and playlist curators. The system processes a contact database, fetches previous email threads to maintain contextual relevance, and creates customized email drafts ready for review and sending in Gmail.

The logic is grouped into the following blocks:

- **1.1 Contact Data Loading and Preparation:** Load contacts and campaign configuration, filter and match contacts by language, and prepare the contact list for processing.
- **1.2 Gmail Conversation Retrieval and Filtering:** For each contact, fetch previous Gmail messages to analyze communication context.
- **1.3 AI Prompt Assembly:** Build system and user prompts combining campaign data, conversation history, and contact category to guide AI email generation.
- **1.4 AI Email Generation:** Generate personalized email subjects and HTML bodies using GPT-5-based language models.
- **1.5 Email Draft Creation:** Create Gmail drafts with the generated content and appropriate signatures for manual review.

---

### 2. Block-by-Block Analysis

#### 2.1 Contact Data Loading and Preparation

- **Overview:** This block loads contact details and campaign configuration, matches contacts by language, and filters inactive or incomplete records.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Contacts  
  - Extract from File  
  - AutomatizationHelper  
  - MatchingByLanguage  
  - Filter  
  - Loop Over Items

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually  
    - Input: None  
    - Output: Triggers "Contacts" and "AutomatizationHelper" nodes simultaneously  
    - Edge cases: None

  - **Contacts**  
    - Type: Read/Write File  
    - Role: Reads local file containing contact database (CSV/Excel)  
    - Config: File path set to "/yourdirectory/yourfile" (placeholder, must be customized)  
    - Input: Trigger  
    - Output: Contact data file stream to "Extract from File"  
    - Edge cases: File missing or inaccessible, invalid format

  - **Extract from File**  
    - Type: Extract From File (XLSX operation)  
    - Role: Parses the contact file into JSON objects, including empty cells, assumes header row  
    - Config: Header row enabled, includes empty cells  
    - Input: File data from "Contacts"  
    - Output: Parsed contacts JSON array to "MatchingByLanguage"  
    - Edge cases: Malformed files, incorrect sheet format

  - **AutomatizationHelper**  
    - Type: Google Sheets  
    - Role: Loads campaign settings, links, prompts, and templates from Google Sheets  
    - Config: Document ID and sheet name set to specific Google Sheet containing campaign data  
    - Input: Trigger  
    - Output: Campaign config JSON to "MatchingByLanguage"  
    - Credential: Google Sheets OAuth2 required  
    - Edge cases: Authentication failure, sheet access issues

  - **MatchingByLanguage**  
    - Type: Merge  
    - Role: Combines contact data with campaign settings filtering by language field  
    - Config: Mode "combine", matches on "LANGUAGE" field  
    - Input: Contacts data and campaign config  
    - Output: Filtered combined data to "Filter"  
    - Edge cases: Missing LANGUAGE fields, mismatched data

  - **Filter**  
    - Type: Filter  
    - Role: Filters contacts based on multiple criteria: matching category, active status (boolean), presence of email, non-empty prompt  
    - Config:  
      - CATEGORY field contains category  
      - Active status true (field ‘ative’ matches ‘Aktivní/neaktivní’)  
      - email_1 not empty  
      - PROMPT not empty  
    - Input: Merged contacts  
    - Output: Valid contacts to "Loop Over Items"  
    - Edge cases: Incorrect field names, data inconsistencies

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes contacts one by one for subsequent steps  
    - Config: Batch size = 1  
    - Input: Filtered contacts  
    - Output: Individual contacts to "No Operation, do nothing" and "PreviousMessagesByContact"  
    - Edge cases: Large datasets may slow processing

#### 2.2 Gmail Conversation Retrieval and Filtering

- **Overview:** For each individual contact, retrieve the latest 5 Gmail messages sent to that contact to analyze prior communication and context.
- **Nodes Involved:**  
  - PreviousMessagesByContact  
  - If  
  - ConvertToText  
  - Default  
  - No Operation, do nothing

- **Node Details:**

  - **PreviousMessagesByContact**  
    - Type: Gmail node (getAll operation)  
    - Role: Retrieves up to 5 recent Gmail conversations sent to the contact's primary email address from the user’s Gmail account  
    - Config: Query filters on `to: email_1` and sender is the authenticated user email ("youremail@gmail.com")  
    - Credential: Gmail OAuth2 required  
    - Input: Single contact item  
    - Output: Message list to "If" node  
    - Edge cases: Gmail API rate limits, auth errors, no messages found

  - **If**  
    - Type: If condition  
    - Role: Checks if any previous messages exist (length > 0)  
    - Input: Messages from "PreviousMessagesByContact"  
    - Output:  
      - True: "ConvertToText"  
      - False: "Default" (fallback messages)  
    - Edge cases: Empty or malformed API response

  - **ConvertToText**  
    - Type: Code  
    - Role: Cleans Gmail HTML message bodies by stripping Gmail signatures and HTML tags, extracts subject and body text  
    - Key logic: Regex to detect Gmail signature div, remove all HTML tags except allowed formatting  
    - Input: Gmail messages  
    - Output: Clean text versions of email content to "SystemPrompt"  
    - Edge cases: Unexpected HTML structures, missing fields

  - **Default**  
    - Type: Code  
    - Role: Supplies default subject and body text if no previous messages exist  
    - Config: Hardcoded default subject and content placeholders  
    - Input: None (triggered by no previous messages)  
    - Output: Default context to "SystemPrompt"

  - **No Operation, do nothing**  
    - Type: NoOp (no operation)  
    - Role: Placeholder for workflow path without Gmail retrieval (currently unused)  
    - Input: From "Loop Over Items" (branch not used further)  
    - Output: None

#### 2.3 AI Prompt Assembly

- **Overview:** Builds the system and user prompts for the AI model combining campaign info, conversation history, contact category, and language preferences to guide email generation.
- **Nodes Involved:**  
  - SystemPrompt  
  - Basic LLM Chain

- **Node Details:**

  - **SystemPrompt**  
    - Type: Code  
    - Role: Constructs a comprehensive system prompt describing email generation rules, formatting, tone, relationship context, and category-specific instructions for GPT-5  
    - Key variables:  
      - Extracts campaign links (LATEST SINGLE, LATEST VIDEO, EPK, ONEPAGER) from AutomatizationHelper data  
      - Compiles previous emails examples (subject and cleaned body) from conversation history  
      - Builds user prompt embedding contact’s prompt, category, language, subject, and venue name  
    - Input: Cleaned Gmail messages or default context  
    - Output: JSON with "systemPrompt" and "UserPrompt" for AI model  
    - Edge cases: Missing campaign links, inconsistent data fields

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Sends the user and system prompts to GPT-5 to generate the personalized email content  
    - Config: Uses defined prompt type "define" with output parser enabled  
    - Input: Prompts from "SystemPrompt"  
    - Output: Raw AI output (structured or unstructured) to "ProcessingOutput"  
    - Edge cases: API failures, timeout, prompt formatting errors  
    - Retry enabled for robustness

#### 2.4 AI Email Generation and Processing

- **Overview:** Parses AI output into structured email components, composes final email content with signature and prepares data for Gmail draft creation.
- **Nodes Involved:**  
  - Structured Output Parser  
  - ProcessingOutput  
  - OpenAI Chat Model1

- **Node Details:**

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (structured)  
    - Role: Validates and extracts AI output fields: "subject" (max 50 chars) and "content" (HTML body) based on JSON schema  
    - Config: Auto-fix enabled, strict schema disallowing additional properties  
    - Input: AI raw output (from "OpenAI Chat Model1")  
    - Output: Parsed structured data to "Basic LLM Chain" for final output generation  
    - Edge cases: Parsing errors, schema mismatches

  - **OpenAI Chat Model1**  
    - Type: LangChain Chat LLM (GPT-5)  
    - Role: Secondary AI call to refine or generate final email content using model "gpt-5-nano-2025-08-07"  
    - Credential: OpenAI API key  
    - Input: Output from "Basic LLM Chain" (chained usage)  
    - Output: Raw AI output to "Structured Output Parser"  
    - Edge cases: API errors, version constraints

  - **ProcessingOutput**  
    - Type: Code  
    - Role: Combines AI-generated subject and content with contact email addresses and signature from campaign config  
    - Input: AI parsed output and contact item data  
    - Output: JSON object with fields: email1, email2, venue_name, subject, body, signature for Gmail draft creation  
    - Edge cases: Missing signature or email fields

#### 2.5 Email Draft Creation

- **Overview:** Uses Gmail API to create an email draft with generated HTML content, including CC and signature, ready for manual review and sending.
- **Nodes Involved:**  
  - Create a draft

- **Node Details:**

  - **Create a draft**  
    - Type: Gmail node (draft creation)  
    - Role: Creates a Gmail draft email with generated subject, HTML body, and adds CC from second email if available  
    - Config:  
      - Subject: AI-generated subject  
      - Message: HTML body + signature  
      - Send To: Primary email (email1)  
      - CC: Secondary email (email2) if present  
    - Credential: Gmail OAuth2  
    - Input: Processed email data from "ProcessingOutput"  
    - Output: Draft creation confirmation and info  
    - Edge cases: Gmail API limits, invalid email addresses, auth failures

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                                | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                   |
|----------------------------|---------------------------------|------------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Entry point to start the workflow               | None                        | Contacts, AutomatizationHelper | ## Short Description: Advanced AI-powered email campaign system for musicians...               |
| Contacts                   | Read/Write File                 | Loads contact database file                      | When clicking ‘Execute workflow’ | Extract from File           | ## CONTACTS Loading contact database from file...                                              |
| Extract from File          | Extract From File (XLSX)         | Parses contacts file into JSON                   | Contacts                    | MatchingByLanguage          |                                                                                                 |
| AutomatizationHelper       | Google Sheets                   | Loads campaign settings and templates            | When clicking ‘Execute workflow’ | MatchingByLanguage          | ## Configuration & Contact Filtering Configuration setup and contact preparation...            |
| MatchingByLanguage         | Merge                          | Matches contacts and campaign config by language | Extract from File, AutomatizationHelper | Filter                    |                                                                                                 |
| Filter                    | Filter                         | Filters contacts by active status, category, and emails | MatchingByLanguage          | Loop Over Items             |                                                                                                 |
| Loop Over Items            | Split In Batches               | Processes each contact individually              | Filter                      | No Operation, do nothing; PreviousMessagesByContact |                                                                                                 |
| No Operation, do nothing  | NoOp                           | Placeholder branch (not used)                     | Loop Over Items             | None                       |                                                                                                 |
| PreviousMessagesByContact | Gmail (getAll)                 | Retrieves last 5 Gmail messages for contact      | Loop Over Items             | If                         | ##1. GMAIL RETRIEVAL: Fetches previous email conversation history...                           |
| If                        | If                             | Checks if previous messages exist                 | PreviousMessagesByContact   | ConvertToText (true), Default (false) |                                                                                                 |
| ConvertToText             | Code                           | Cleans Gmail HTML messages to plain text         | If (true)                   | SystemPrompt               |                                                                                                 |
| Default                   | Code                           | Provides default email content if no history      | If (false)                  | SystemPrompt               |                                                                                                 |
| SystemPrompt              | Code                           | Builds AI system and user prompts                  | ConvertToText, Default      | Basic LLM Chain            | ## PROMPT ASSEMBLY: Builds system prompt with campaign info...                                 |
| Basic LLM Chain           | LangChain LLM Chain            | Sends prompts to GPT-5 for email generation       | SystemPrompt                | ProcessingOutput           | ## AI GENERATION: LLM generates personalized email content...                                 |
| ProcessingOutput          | Code                           | Combines AI output with contact info and signature | Basic LLM Chain            | Create a draft             |                                                                                                 |
| Create a draft            | Gmail (draft creation)         | Creates Gmail draft with personalized email       | ProcessingOutput            | Loop Over Items            | ## GMAIL DRAFT: Creates email draft in Gmail with generated content...                         |
| OpenAI Chat Model         | LangChain Chat LLM (GPT-5)     | GPT-5 model for initial AI output generation      | Basic LLM Chain             | Structured Output Parser   |                                                                                                 |
| Structured Output Parser  | LangChain Output Parser        | Parses AI output into structured subject and content | OpenAI Chat Model1          | Basic LLM Chain            |                                                                                                 |
| OpenAI Chat Model1        | LangChain Chat LLM (GPT-5)     | GPT-5 model for refined output                     | Basic LLM Chain             | Structured Output Parser   |                                                                                                 |
| Sticky Note               | Sticky Note                    | Notes describing contacts loading                 | None                       | None                      | ## CONTACTS Loading contact database from file...                                              |
| Sticky Note1              | Sticky Note                    | Notes on configuration and filtering               | None                       | None                      | ## Configuration & Contact Filtering Configuration setup and contact preparation...            |
| Sticky Note2              | Sticky Note                    | Notes on Gmail retrieval                            | None                       | None                      | ##1. GMAIL RETRIEVAL: Fetches previous email conversation history...                           |
| Sticky Note3              | Sticky Note                    | Notes on prompt assembly                            | None                       | None                      | ## PROMPT ASSEMBLY: Builds system prompt with campaign info...                                 |
| Sticky Note4              | Sticky Note                    | Notes on AI generation                              | None                       | None                      | ## AI GENERATION: LLM generates personalized email content...                                 |
| Sticky Note5              | Sticky Note                    | Notes on Gmail draft creation                       | None                       | None                      | ## GMAIL DRAFT: Creates email draft in Gmail with generated content...                         |
| Sticky Note6              | Sticky Note                    | Overview note summarizing workflow                 | None                       | None                      | ## Short Description: Advanced AI-powered email campaign system for musicians...               |
| Sticky Note7              | Sticky Note                    | Indicates branch where drafts are already generated | None                       | None                      | ## Done Branch Drafts have been already generated                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Manually start workflow execution  
   - Connect output to "Contacts" and "AutomatizationHelper"

2. **Create Contacts Node (Read/Write File)**  
   - Type: Read/Write File  
   - Configure file selector to point to your local contact file (CSV/XLSX) containing email, name, category, status, language, etc.

3. **Create Extract from File Node**  
   - Type: Extract From File  
   - Operation: XLSX  
   - Options: Enable header row, include empty cells  
   - Connect input from "Contacts"  
   - Output to "MatchingByLanguage"

4. **Create AutomatizationHelper Node (Google Sheets)**  
   - Type: Google Sheets  
   - Configure with Google Sheets OAuth2 credentials  
   - Set Document ID and Sheet Name to your campaign settings spreadsheet  
   - Output to "MatchingByLanguage"

5. **Create MatchingByLanguage Node (Merge)**  
   - Mode: Combine  
   - Fields to match on: LANGUAGE  
   - Inputs: Connect from "Extract from File" and "AutomatizationHelper"  
   - Output to "Filter"

6. **Create Filter Node**  
   - Conditions (AND):  
     - CATEGORY contains category field  
     - Active status field equals true (map fields accordingly)  
     - email_1 is not empty  
     - PROMPT is not empty  
   - Input: "MatchingByLanguage"  
   - Output: "Loop Over Items"

7. **Create Loop Over Items Node (Split In Batches)**  
   - Batch size: 1  
   - Input: "Filter"  
   - Outputs:  
     - Connect one output to "No Operation, do nothing" (optional)  
     - Connect second output to "PreviousMessagesByContact"

8. **Create PreviousMessagesByContact Node (Gmail)**  
   - Operation: getAll  
   - Limit: 5  
   - Filters: Query `to: {{ $json["email_1"] }}`, sender your Gmail  
   - Credentials: Gmail OAuth2  
   - Input: "Loop Over Items"  
   - Output: "If"

9. **Create If Node**  
   - Condition: Check if length of messages > 0  
   - True output: "ConvertToText"  
   - False output: "Default"

10. **Create ConvertToText Node (Code)**  
    - JavaScript code to strip Gmail signatures and HTML tags, return cleaned subject and body  
    - Input: "If" (true branch)  
    - Output: "SystemPrompt"

11. **Create Default Node (Code)**  
    - JavaScript code returning default subject and email content placeholders  
    - Input: "If" (false branch)  
    - Output: "SystemPrompt"

12. **Create SystemPrompt Node (Code)**  
    - JavaScript code assembling system prompt with campaign links, previous email examples, contact category, language, and user prompt  
    - Input: "ConvertToText" and "Default"  
    - Output: "Basic LLM Chain"

13. **Create Basic LLM Chain Node (LangChain)**  
    - Type: Chain LLM  
    - Model: GPT-5 (via LangChain)  
    - Parameters: Pass system and user prompts, enable output parser  
    - Retry on fail enabled  
    - Input: "SystemPrompt"  
    - Output: "ProcessingOutput"

14. **Create OpenAI Chat Model1 Node (LangChain)**  
    - Model: GPT-5 Nano (2025-08-07)  
    - Credentials: OpenAI API key  
    - Input: "Basic LLM Chain" (for refinement)  
    - Output: "Structured Output Parser"

15. **Create Structured Output Parser Node (LangChain)**  
    - Schema: JSON object with "subject" (max 50 chars) and "content" (HTML)  
    - Auto-fix enabled  
    - Input: "OpenAI Chat Model1"  
    - Output: "Basic LLM Chain" (feedback loop)

16. **Create ProcessingOutput Node (Code)**  
    - Combines AI output with contact emails and signature from campaign config  
    - Returns JSON with email1, email2, subject, body, signature  
    - Input: "Basic LLM Chain"  
    - Output: "Create a draft"

17. **Create Create a draft Node (Gmail)**  
    - Operation: Draft creation  
    - Subject: `={{ $json.subject }}`  
    - Message: `={{ $json.body }}{{ $json.signature }}` (HTML)  
    - Send To: `={{ $json.email1 }}`  
    - CC: `={{ $json.email2 }}` (optional)  
    - Credentials: Gmail OAuth2  
    - Input: "ProcessingOutput"  
    - Output: Connect to "Loop Over Items" for batch continuation

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow uses Gmail OAuth2 and Google Sheets OAuth2 credentials; ensure these are properly configured in n8n before use.           | Credential setup                                                                                                                          |
| Gmail signatures are not automatically inserted by Gmail API; signature must be appended manually from campaign spreadsheet data. | See "AutomatizationHelper" node and email signature handling                                                                                |
| Campaign configuration and prompts are loaded dynamically from Google Sheets to allow easy updates without workflow changes.       | See AutomatizationHelper node                                                                                                             |
| AI model used is GPT-5 "gpt-5-nano" and "gpt-5-nano-2025-08-07", accessed via LangChain integration in n8n.                       | Requires valid OpenAI API key and LangChain nodes                                                                                          |
| The workflow supports multi-language email generation and respects tone, greeting style, and relationship level for personalization.| Described in SystemPrompt construction                                                                                                   |
| For best results, ensure contact database contains accurate fields: email addresses, active status, category, language preference.| Refer to Sticky Note content for required contact file structure                                                                           |
| Gmail API rate limits and OpenAI API quotas may affect workflow execution speed and reliability.                                   | Consider error handling and retry policies                                                                                                 |
| No placeholders are used in generated emails; all variables are resolved during prompt assembly to avoid sending incomplete emails.| SystemPrompt enforces this rule                                                                                                           |
| Draft emails are created for manual review before sending to prevent accidental sending of incorrect content.                      | Workflow does not send emails automatically                                                                                                |
| Sticky Notes in the workflow visually document each block function and usage instructions.                                         | See nodes named Sticky Note, Sticky Note1, etc.                                                                                            |

---

This completes the detailed reference document enabling reproduction, modification, and integration troubleshooting for the "Generate Personalized Music Promotion Emails with GPT-5 and Gmail Context Analysis" workflow.