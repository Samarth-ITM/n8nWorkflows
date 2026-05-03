Autonomous Email Management with GPT-5-mini & Human-in-the-Loop for Outlook

https://n8nworkflows.xyz/workflows/autonomous-email-management-with-gpt-5-mini---human-in-the-loop-for-outlook-9128


# Autonomous Email Management with GPT-5-mini & Human-in-the-Loop for Outlook

### 1. Workflow Overview

This workflow, titled **"Autonomous Email Management with GPT-5-mini & Human-in-the-Loop for Outlook"**, is a comprehensive AI-powered email automation system designed for Microsoft Outlook users. It autonomously processes incoming emails by classifying them into predefined categories, generating personalized responses using a brand-consistent voice, handling urgent messages with human oversight, managing meeting scheduling, organizing the inbox by auto-archiving, tagging processed emails, and maintaining a detailed audit trail in Excel.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Information Extraction**  
  Captures new incoming emails via Outlook trigger and extracts essential metadata and sender details for processing.

- **1.2 AI Classification Engine**  
  Applies a dual AI-based classification system to categorize emails into seven key categories, enabling targeted processing.

- **1.3 Routing and Conditional Processing**  
  Routes emails to different processing branches based on classification results, including urgent handling, meeting scheduling, general replies, and "Other" category filtering.

- **1.4 AI Response Generation (Brand Voice System)**  
  Generates email replies using specialized AI prompt chains tailored for each category, ensuring consistent, professional communication in the user's brand voice.

- **1.5 Meeting Management**  
  For meeting-related emails, integrates with calendar tools to check availability, propose alternative timings, and create events, respecting buffer times and working hours.

- **1.6 Urgent Email Handling with Human-in-the-Loop**  
  Sends Slack notifications for urgent emails, drafts prompt AI replies, and awaits manual approval before sending.

- **1.7 Inbox Organization and Tagging**  
  Marks emails as read, tags them with "AI" for tracking, and moves them to category-specific Outlook folders.

- **1.8 Data Logging and Audit Trail**  
  Logs all processed emails and generated replies into a Microsoft Excel 365 workbook for compliance, analytics, and quality assurance.

- **1.9 Feedback Loop for Draft Revision**  
  Allows human feedback on drafts for urgent emails with AI-assisted revision before final sending.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Information Extraction

- **Overview:**  
  Captures incoming Outlook emails and extracts sender information, including the first name, to personalize responses.

- **Nodes Involved:**  
  - Microsoft Outlook Trigger1  
  - OpenRouter Chat Model2  
  - Information Extractor1  
  - Virtual Postman1  
  - Edit Fields4, Edit Fields5, Edit Fields6, Edit Fields7

- **Node Details:**  
  - **Microsoft Outlook Trigger1**  
    - Type: Microsoft Outlook email trigger  
    - Role: Listens for new emails in inbox, polling every minute  
    - Credentials: Microsoft Outlook OAuth2 (Personal Outlook)  
    - Edge Cases: Trigger delays or duplicate triggers if polling interval misconfigured; network errors during polling.

  - **OpenRouter Chat Model2**  
    - Type: AI Language Model (OpenRouter GPT-5-mini)  
    - Role: Provides AI-based NLP processing support for downstream nodes  
    - Credentials: OpenRouter API  
    - Edge Cases: API rate limits, timeouts, invalid credentials.

  - **Information Extractor1**  
    - Type: Langchain Information Extractor  
    - Role: Parses email metadata and extracts sender's first name for personalization  
    - Input: Email subject, sender, body preview, email ID  
    - Output: sender_name attribute  
    - Edge Cases: Missing or malformed sender info, extraction failures.

  - **Virtual Postman1**  
    - Type: Langchain Text Classifier  
    - Role: Classifies email into one of seven predefined categories  
    - Input: Email metadata and body preview  
    - Categories: Commercial/Spam, Internal, Meeting, Newsletter, Notifications, Urgent, Other  
    - Edge Cases: Ambiguous emails, misclassification.

  - **Edit Fields4, Edit Fields5, Edit Fields6, Edit Fields7**  
    - Type: Set nodes  
    - Role: Prepare and standardize fields for downstream processing (e.g., id, subject, bodyPreview, from, sender_name)  
    - Edge Cases: Missing data, expression evaluation errors.

---

#### 2.2 AI Classification Engine

- **Overview:**  
  Applies a two-tier classification: primary AI-driven text classification (Virtual Postman) and downstream conditional routing for precise categorization and processing logic.

- **Nodes Involved:**  
  - Virtual Postman1 (also part of 2.1)  
  - Classifier2  
  - Classifier3

- **Node Details:**  
  - **Virtual Postman1**  
    - See above.

  - **Classifier2**  
    - Type: Langchain Text Classifier  
    - Role: Classifies AI Agent response content into "Confirming" or "Suggesting" for meeting scheduling decisions  
    - Input: AI-generated message content  
    - Edge Cases: Misinterpretation of confirmation intent.

  - **Classifier3**  
    - Type: Langchain Text Classifier  
    - Role: Classifies "Other" category emails into "Requires reply" or "Doesn't require reply" to decide if follow-up is needed  
    - Input: AI-generated message content  
    - Edge Cases: Misclassification causing unnecessary replies or missed responses.

---

#### 2.3 Routing and Conditional Processing

- **Overview:**  
  Routes emails to specific processing branches based on classification: urgent emails get special handling; meeting emails are sent to scheduling logic; others processed accordingly.

- **Nodes Involved:**  
  - Branching occurs via node connections from Virtual Postman1 and classification nodes (Classifier2, Classifier3).

- **Node Details:**  
  - Routing is implicit through connected nodes after classification. For instance, urgent emails lead to urgentReplier1 and Slack1; meeting emails flow to AI Agent1; others flow to emailReplier1 or further classification.

  - Edge Cases: Incorrect routing due to misclassification; missing fallback or error handling nodes.

---

#### 2.4 AI Response Generation (Brand Voice System)

- **Overview:**  
  Generates concise, professional email replies tailored to each category, enforcing a consistent brand voice using chained LLM nodes.

- **Nodes Involved:**  
  - AI Agent1  
  - emailReplier1  
  - urgentReplier1  
  - Brand Voice4, Brand Voice5, Brand Voice6, Brand Voice7  
  - Revise Draft (chainLlm for draft revision)

- **Node Details:**  
  - **AI Agent1**  
    - Type: Langchain Agent  
    - Role: Handles meeting-related emails with calendar integration tools  
    - Configuration: Custom system message instructing scheduling logic, buffer times, and working hours  
    - Edge Cases: Calendar API failures, inconsistent availability data.

  - **emailReplier1**  
    - Type: Langchain Chain LLM  
    - Role: Generates replies for general email categories  
    - Prompt: Prioritizes information requests, clarifications, task delegation, or acknowledgments  
    - Edge Cases: Ambiguous email content leading to irrelevant replies.

  - **urgentReplier1**  
    - Type: Langchain Chain LLM  
    - Role: Crafts prompt and professional replies for urgent emails  
    - Edge Cases: Misinterpretation of urgency or incomplete information.

  - **Brand Voice4, Brand Voice5, Brand Voice6, Brand Voice7**  
    - Type: Langchain Chain LLM  
    - Role: Rewrites AI-generated drafts into Didac’s direct, concise, professional style with specific phrasing and sign-off  
    - Edge Cases: LLM output inconsistencies, API errors.

  - **Revise Draft**  
    - Type: Langchain Chain LLM  
    - Role: Revises draft emails based on human feedback for urgent messages  
    - Input: Original draft, user feedback, email metadata  
    - Edge Cases: Misinterpretation of feedback, incorrect revisions.

---

#### 2.5 Meeting Management

- **Overview:**  
  Uses AI and Microsoft Outlook calendar tools to automate meeting scheduling, including availability checks, suggesting alternatives, and event creation.

- **Nodes Involved:**  
  - AI Agent1  
  - eventChecker1  
  - eventCreator1  
  - Window Buffer Memory1  
  - Classifier2  
  - confirmEvent1  
  - proposeEvent1

- **Node Details:**  
  - **eventChecker1**  
    - Type: Microsoft Outlook Tool  
    - Role: Checks calendar availability for suggested meeting times  
    - Credentials: Microsoft Outlook OAuth2  
    - Edge Cases: API timeouts, calendar sync issues.

  - **eventCreator1**  
    - Type: Microsoft Outlook Tool  
    - Role: Creates calendar events upon confirmation  
    - Edge Cases: Conflicting events, permissions issues.

  - **Window Buffer Memory1**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains conversation context for AI Agent during scheduling  
    - Edge Cases: Loss of context, memory overflow.

  - **Classifier2**  
    - Determines if the user is confirming a proposed time or suggesting alternatives.

  - **confirmEvent1**  
    - Type: Microsoft Outlook node  
    - Role: Moves confirmed meeting emails to Event folder  
    - Edge Cases: Folder permission issues.

  - **proposeEvent1**  
    - Type: Microsoft Outlook node  
    - Role: Replies to meeting proposal emails with suggested times  
    - Edge Cases: Reply failures.

---

#### 2.6 Urgent Email Handling with Human-in-the-Loop

- **Overview:**  
  Urgent emails trigger Slack notifications and AI draft replies that require manual approval before sending, providing a human oversight layer.

- **Nodes Involved:**  
  - urgentReplier1  
  - Brand Voice6  
  - Slack1  
  - Microsoft Outlook1  
  - Check Feedback1  
  - Revise Draft

- **Node Details:**  
  - **Slack1**  
    - Type: Slack node  
    - Role: Sends direct message notification to user with email details and draft alert  
    - Credentials: Slack OAuth2  
    - Edge Cases: Slack API rate limits, network issues.

  - **Microsoft Outlook1**  
    - Type: Microsoft Outlook email sender with response type "freeText"  
    - Role: Sends approved replies post human review  
    - Edge Cases: Sending failures, invalid email addresses.

  - **Check Feedback1**  
    - Type: Langchain Text Classifier  
    - Role: Classifies human feedback as Approved or Declined to control send or revision flow  
    - Edge Cases: Misclassification causing unintended sends or loops.

  - **Revise Draft**  
    - See above.

---

#### 2.7 Inbox Organization and Tagging

- **Overview:**  
  After processing, emails are marked as read, tagged with category "AI", and moved to category-specific folders in Outlook for orderly management.

- **Nodes Involved:**  
  - markedAsRead3, markedAsRead7, markedAsRead9, markedAsRead10, markedAsRead11, markedAsRead12, markedAsRead13  
  - aiTagging, aiTagging8, aiTagging9, aiTagging10, aiTagging11, aiTagging12, aiTagging13  
  - archiveCommercialSpam1, archiveNewsletter1, archiveNotifications1, archiveEvent1, archiveInternal1, archiveUrgent1, archiveNotifications7

- **Node Details:**  
  - **markedAsReadX nodes**  
    - Type: Microsoft Outlook update nodes  
    - Role: Mark respective emails as read to avoid duplicate processing  
    - Edge Cases: Update failures.

  - **aiTaggingX nodes**  
    - Type: Microsoft Outlook update nodes  
    - Role: Add "AI" category tag to processed emails for tracking  
    - Edge Cases: Tagging failures, permission issues.

  - **archiveX nodes**  
    - Type: Microsoft Outlook move operations  
    - Role: Move emails into folders matching their classification  
    - Folder IDs hardcoded for categories (e.g., Commercial/Spam, Newsletter)  
    - Edge Cases: Folder permission or existence issues.

---

#### 2.8 Data Logging and Audit Trail

- **Overview:**  
  Logs key email metadata and generated replies into an Excel workbook for auditing, analytics, and compliance.

- **Nodes Involved:**  
  - Microsoft Excel 3651

- **Node Details:**  
  - **Microsoft Excel 3651**  
    - Type: Microsoft Excel node  
    - Role: Appends processed email data (ID, Date, Subject, Sender, Body preview, Reply) into "Email Automator" workbook, Sheet1  
    - Credentials: Microsoft Excel OAuth2  
    - Edge Cases: API limits, file access permissions, workbook availability.

---

#### 2.9 Feedback Loop for Draft Revision

- **Overview:**  
  Allows human reviewers to provide feedback on AI-generated drafts for urgent emails, with AI revising drafts accordingly before final sending.

- **Nodes Involved:**  
  - Check Feedback1  
  - Revise Draft  
  - Microsoft Outlook1

- **Node Details:**  
  - See details under urgent email handling and AI response generation above.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                                     | Input Node(s)               | Output Node(s)                | Sticky Note                                      |
|-----------------------|----------------------------------|----------------------------------------------------|-----------------------------|------------------------------|-------------------------------------------------|
| Microsoft Outlook Trigger1 | Microsoft Outlook Trigger          | Incoming email listener                             | -                           | Information Extractor1        | # Autonomous Email Assistant                      |
| OpenRouter Chat Model2 | Langchain LM Chat OpenRouter      | AI language model for multiple AI nodes            | -                           | Information Extractor1, Virtual Postman1, AI Agent1, Brand Voice*, Classifier2, emailReplier1, urgentReplier1, Revise Draft, Classifier3, Brand Voice7 | # Autonomous Email Assistant                      |
| Information Extractor1 | Langchain Information Extractor   | Extract sender first name                           | Microsoft Outlook Trigger1   | Virtual Postman1              | ## INFORMATION EXTRACTOR                          |
| Virtual Postman1       | Langchain Text Classifier         | Primary AI classification of emails                | Information Extractor1       | Multiple Edit Fields, markedAsRead nodes          | ## MESSAGE TRIAGE AND CATEGORIZATION             |
| Edit Fields4           | Set                              | Prepare fields for meeting processing               | Virtual Postman1             | AI Agent1                    | ## TEXT MOVER                                     |
| Edit Fields5           | Set                              | Prepare fields for internal emails                  | Virtual Postman1             | emailReplier1                | ## TEXT MOVER                                     |
| Edit Fields6           | Set                              | Prepare fields for urgent emails                     | Virtual Postman1             | urgentReplier1               | ## TEXT MOVER                                     |
| Edit Fields7           | Set                              | Prepare fields for "Other" category                  | Virtual Postman1             | Classifier3                  | ## TEXT MOVER                                     |
| AI Agent1              | Langchain Agent                   | Meeting scheduling AI with calendar integration    | Edit Fields4                 | Brand Voice4                 | ## AI EMAIL COMPOSER                              |
| Brand Voice4           | Langchain Chain LLM              | Brand voice rewriting for meetings                   | AI Agent1                   | Classifier2                  | ## BRAND VOICE                                    |
| Classifier2            | Langchain Text Classifier        | Classify meeting replies as Confirming or Suggesting| Brand Voice4                 | confirmEvent1, proposeEvent1 | ## FLOW CLASSIFIER                                |
| confirmEvent1          | Microsoft Outlook                | Move confirmed meeting emails to Event folder       | Classifier2                 | markedAsRead10               | ## EMAIL ARCHIVE IN FOLDER                        |
| proposeEvent1          | Microsoft Outlook                | Reply with meeting proposals                          | Classifier2                 | markedAsRead10               | ## EMAIL ARCHIVE IN FOLDER                        |
| emailReplier1          | Langchain Chain LLM              | Generate replies for general/internal emails         | Edit Fields5                 | Brand Voice5                 | ## AI EMAIL COMPOSER                              |
| Brand Voice5           | Langchain Chain LLM              | Brand voice rewriting for general replies            | emailReplier1               | draftReply1                  | ## BRAND VOICE                                    |
| draftReply1            | Microsoft Outlook                | Save general replies as draft                          | Brand Voice5                 | markedAsRead12               | ## EMAIL DRAFTER                                  |
| urgentReplier1         | Langchain Chain LLM              | Generate prompt replies for urgent emails             | Edit Fields6                 | Brand Voice6                 | ## AI EMAIL COMPOSER                              |
| Brand Voice6           | Langchain Chain LLM              | Brand voice rewriting for urgent replies              | urgentReplier1              | Slack1, Microsoft Outlook1   | ## BRAND VOICE                                    |
| Slack1                 | Slack                           | Notify user of urgent email and draft ready          | Brand Voice6                | -                           | ## DIRECT MESSAGE ON SLACK                        |
| Microsoft Outlook1     | Microsoft Outlook                | Send final approved urgent replies                    | Brand Voice6, Revise Draft  | Check Feedback1              | ## EMAIL SEND & FEEDBACK                          |
| Check Feedback1        | Langchain Text Classifier        | Classify human feedback on drafts (Approve/Decline) | Microsoft Outlook1           | markedAsRead11, Revise Draft | ## FLOW CLASSIFIER                                |
| Revise Draft           | Langchain Chain LLM              | AI revises draft based on feedback                     | Check Feedback1             | Microsoft Outlook1           | ## AI EMAIL COMPOSER                              |
| Classifier3            | Langchain Text Classifier        | For "Other" emails, determine if reply is needed      | Edit Fields7                 | Brand Voice7, markedAsRead13 | ## FLOW CLASSIFIER                                |
| Brand Voice7           | Langchain Chain LLM              | Brand voice rewriting for "Other" category replies    | Classifier3                 | draftReply3                  | ## BRAND VOICE                                    |
| draftReply3            | Microsoft Outlook                | Save "Other" replies as draft                           | Brand Voice7                 | markedAsRead13               | ## EMAIL DRAFTER                                  |
| markedAsRead3/7/9/10/11/12/13 | Microsoft Outlook update      | Mark emails as read in Outlook                         | Various classification nodes| aiTagging nodes              | ## EMAIL READ                                     |
| aiTagging/aiTagging8-13| Microsoft Outlook update        | Tag emails with "AI" category                          | markedAsRead nodes           | archive nodes                | ## EMAIL TAGGED AS "AI"                           |
| archiveCommercialSpam1, archiveNewsletter1, archiveNotifications1, archiveEvent1, archiveInternal1, archiveUrgent1, archiveNotifications7 | Microsoft Outlook move | Move emails to category-specific folders              | aiTagging nodes             | Microsoft Excel 3651                               | ## EMAIL ARCHIVE IN FOLDER                        |
| Microsoft Excel 3651   | Microsoft Excel                  | Log email metadata and replies for auditing           | archive nodes               | -                           | ## BODY SAVED ON CSV FOR FUTURE MEMORY           |
| Window Buffer Memory1  | Langchain Memory Buffer Window  | Maintain AI conversation context for scheduling       | Edit Fields4                | AI Agent1                   | ## AI EMAIL COMPOSER                              |
| eventChecker1          | Microsoft Outlook Tool          | Check calendar availability                            | AI Agent1 (ai_tool)         | AI Agent1                   | ## MEETING REQUEST HANDLER                        |
| eventCreator1          | Microsoft Outlook Tool          | Create calendar event                                  | AI Agent1 (ai_tool)         | AI Agent1                   | ## MEETING REQUEST HANDLER                        |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Credentials**  
- Microsoft Outlook OAuth2 (Personal Outlook account with read, write, send, and calendar access)  
- Microsoft Excel OAuth2 (Personal Office account with access to "Email Automator" workbook)  
- OpenRouter API credentials for GPT-5-mini model  
- Slack OAuth2 for user DM (e.g., user ID U060FTJ71C7)

**Step 2: Create Microsoft Outlook Trigger Node**  
- Type: Microsoft Outlook Trigger  
- Poll every minute  
- Connect credentials (Outlook OAuth2)  
- Output: New email messages

**Step 3: Add OpenRouter Chat Model Node**  
- Type: Langchain LM Chat OpenRouter  
- Model: openai/gpt-5-mini  
- Connect to OpenRouter credentials  
- Output: Feed to AI-based nodes below

**Step 4: Extract Sender Name**  
- Node: Langchain Information Extractor  
- Input text combining subject, sender, body preview, and email ID from Outlook Trigger  
- Attribute to extract: sender_name (first name of sender)  
- Connect output to Virtual Postman classifier

**Step 5: Classify Incoming Email**  
- Node: Langchain Text Classifier (Virtual Postman1)  
- Input: Same combined email text  
- Categories: Commercial/Spam, Internal, Meeting, Newsletter, Notifications, Urgent, Other (with detailed descriptions)  
- Output: Routes to respective processing branches

**Step 6: Setup Field Preparation Nodes**  
- Create four Set nodes (Edit Fields4-7) to normalize and prepare email fields (id, subject, bodyPreview, from, sender_name) for different branches (meetings, internal, urgent, other)  
- Connect Virtual Postman outputs accordingly

**Step 7: Configure Meeting Scheduling Branch**  
- Create Langchain Agent node (AI Agent1) with system message for scheduling logic, integrating eventChecker and eventCreator tools  
- Connect eventChecker1 (Outlook Tool) to check calendar availability  
- Connect eventCreator1 (Outlook Tool) to create events  
- Use Window Buffer Memory node to maintain conversation context  
- Pass AI Agent1 output to Brand Voice4 (Langchain Chain LLM) for style rewriting  
- Connect Brand Voice4 to Classifier2 (Confirming/Suggesting)  
- Setup confirmEvent1 and proposeEvent1 Outlook nodes to move or reply accordingly  
- Mark emails as read and tag with "AI" after processing  
- Archive emails to Event folder  
- Log data to Excel node

**Step 8: Configure General/Internal Email Reply Branch**  
- Connect Edit Fields5 to emailReplier1 (Langchain Chain LLM) for reply generation  
- Connect emailReplier1 to Brand Voice5 for style rewriting  
- Connect Brand Voice5 to draftReply1 Outlook node to save reply as draft  
- Mark email as read, tag as "AI"  
- Archive to Internal folder  
- Log to Excel

**Step 9: Configure Urgent Email Handling Branch**  
- Connect Edit Fields6 to urgentReplier1 (Langchain Chain LLM) for urgent reply generation  
- Connect urgentReplier1 to Brand Voice6 for style rewriting  
- Connect Brand Voice6 to Slack1 node to send Slack notification  
- Also connect Brand Voice6 to Microsoft Outlook1 node configured for sending emails with freeText response type to await human feedback  
- Connect Microsoft Outlook1 to Check Feedback1 (Langchain Text Classifier) to classify feedback as Approved or Declined  
- Approved branch: mark email as read, tag as "AI", archive to Urgent folder, log to Excel  
- Declined branch: connect to Revise Draft node (Langchain Chain LLM) for AI draft revision based on feedback  
- Connect Revise Draft output back to Microsoft Outlook1 for sending updated draft

**Step 10: Configure "Other" Email Branch**  
- Connect Edit Fields7 to Classifier3 (Langchain Text Classifier) to decide if reply is needed  
- If reply required: pass to Brand Voice7 for style rewriting, then draftReply3 (Outlook) to save draft  
- If no reply: mark as read, tag as "AI", archive to Other folder  
- Log all processed emails to Excel

**Step 11: Setup Mark As Read, Tagging, and Archiving Nodes**  
- For each category, create Microsoft Outlook nodes to mark the email as read after processing  
- Tag the email with category "AI" for tracking  
- Move email to the corresponding Outlook folder (Commercial/Spam, Newsletter, Notifications, Event, Internal, Urgent, Other) using folder IDs  
- Connect each archive node to Microsoft Excel 3651 node for logging

**Step 12: Configure Microsoft Excel 365 Node**  
- Setup to append rows to "Email Automator" workbook, Sheet1  
- Columns: Id, Date, Subject, Sender, Body, Reply  
- Map values from processed email and AI-generated reply

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The extensive brand voice prompt embedded in Brand Voice nodes ensures consistent, direct, concise, and professional replies using key phrases like "Just checking", "For your awareness", "Let me know", and the sign-off "Regards, Didac". | Embedded in Brand Voice nodes documentation |
| Microsoft Outlook trigger is set to poll every minute but can be adjusted based on inbox volume and latency requirements. | Microsoft Outlook Trigger1 configuration |
| Folder IDs for archiving are hardcoded Outlook folder identifiers. Adjust these IDs to match your Outlook mailbox folder structure. | Archive nodes configuration |
| Slack notifications are sent to user ID U060FTJ71C7 (@didac). Update Slack node to target the appropriate user or channel. | Slack1 node configuration |
| Human-in-the-loop control for urgent emails ensures accountability and quality control before sending sensitive or time-critical replies. | Urgent email handling section |
| Excel logging provides a comprehensive audit trail and supports analytics, compliance, and training data collection. | Microsoft Excel 3651 node |
| AI model used is OpenRouter GPT-5-mini; ensure API credentials are set and rate limits monitored. | OpenRouter Chat Model2 and OpenRouter Chat Model3 nodes |
| The meeting scheduling AI respects working hours (8:30 AM - 5:00 PM) and maintains a 15-minute buffer between events to prevent overlaps. | AI Agent1 system message instructions |
| Monitoring and maintenance recommendations include reviewing Excel logs weekly, checking AI-tagged emails, and updating classification categories or brand voice prompts as needed. | Sticky Note24 - Tips & Best Practices |
| Safety features include drafts created by default to avoid accidental sends, urgent emails requiring manual approval, and comprehensive logging. | Sticky Note24 - Safety Features |

---

**Disclaimer:**  
The provided documentation is generated from an automated n8n workflow. All components and data are compliant with current content policies and legal standards. The workflow manipulates only lawful and publicly available data.