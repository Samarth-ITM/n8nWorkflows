Send AI job application auto-replies with Gmail, OpenAI GPT-4o and SMTP

https://n8nworkflows.xyz/workflows/send-ai-job-application-auto-replies-with-gmail--openai-gpt-4o-and-smtp-14267


# Send AI job application auto-replies with Gmail, OpenAI GPT-4o and SMTP

# 1. Workflow Overview

This workflow automatically sends AI-generated acknowledgment emails to job applicants. It monitors a Gmail inbox for forwarded hiring emails, filters messages that look like job applications, prepares a small set of fields for AI processing, uses GPT-4o to generate a structured reply, and sends that reply through SMTP from the company email address.

Typical use cases:
- Auto-replying to inbound job applications
- Handling applications received on a custom domain email that forwards into Gmail
- Standardizing first-touch HR responses with minimal manual effort

## 1.1 Input Reception and Mail Intake
The workflow starts with a Gmail Trigger that watches a specific Gmail label. The intended setup is that job application emails sent to a company domain are forwarded into Gmail and labeled there.

## 1.2 Email Qualification
An If node checks whether the subject contains the phrase “job application”. Matching emails proceed to AI processing. Non-matching emails are ignored.

## 1.3 Data Preparation
A Set/Edit Fields node extracts the candidate email, original subject, and company email into a simplified structure for downstream use.

## 1.4 AI Reply Generation
An AI Agent node uses an OpenAI Chat Model and a Structured Output Parser to generate a JSON reply with:
- `subject`
- `body`

The prompt instructs the model to write a professional HR response and return only structured output.

## 1.5 Email Delivery
An Email Send node sends the generated reply to the candidate using SMTP, with the sender set to the company email.

## 1.6 Ignore Path
A No Operation node terminates the branch for emails that do not match the job-application condition.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Mail Intake

### Overview
This block listens for incoming Gmail messages that have already been filtered and labeled as job applications. It is the workflow entry point and depends on correct Gmail forwarding and filtering outside n8n.

### Nodes Involved
- Gmail Trigger

### Node Details

#### Gmail Trigger
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger node that checks Gmail for new matching messages.
- **Configuration choices:**
  - Uses Gmail OAuth2 credentials
  - `simple` mode is disabled, so more complete Gmail message data is available
  - Filters by a specific Gmail label ID: `Label_2862649459238172600`
  - Polling frequency is set to every minute
- **Key expressions or variables used:**  
  None in the node configuration itself.
- **Input and output connections:**
  - Input: none, this is the entry point
  - Output: `If`
- **Version-specific requirements:**
  - Type version: `1.3`
  - Requires a valid Gmail OAuth2 credential in n8n
- **Edge cases or potential failure types:**
  - Gmail OAuth expiration or revoked consent
  - Incorrect label ID or missing label
  - Forwarded emails not being labeled correctly
  - Polling delays or missed assumptions if Gmail marks messages read/archives them before trigger logic can detect them
  - Data shape may differ depending on the incoming email headers and Gmail parsing
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
This node is documented as requiring external forwarding from a custom-domain inbox to Gmail, plus a Gmail filter that applies a dedicated label, keeps the message unread, and avoids spam classification. This setup is essential for reliable triggering.

---

## 2.2 Email Qualification

### Overview
This block determines whether an incoming message should be treated as a job application. Only messages whose subject contains the target phrase are sent to the AI branch.

### Nodes Involved
- If
- No Operation, do nothing

### Node Details

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional routing node.
- **Configuration choices:**
  - Uses a boolean expression
  - Checks whether the subject, converted to lowercase, includes `"job application"`
  - Strict type validation enabled in the condition settings
- **Key expressions or variables used:**
  - `{{$json.subject.toLowerCase().includes("job application")}}`
- **Input and output connections:**
  - Input: `Gmail Trigger`
  - True output: `Edit Fields`
  - False output: `No Operation, do nothing`
- **Version-specific requirements:**
  - Type version: `2.3`
- **Edge cases or potential failure types:**
  - If `subject` is null or undefined, `.toLowerCase()` can fail
  - Subject matching is simplistic and may miss variants such as:
    - “Application for”
    - “CV Submission”
    - localized subject lines
  - False positives are possible if unrelated emails contain the phrase
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
The note explicitly suggests customizing the subject filter to match the organization’s hiring patterns.

#### No Operation, do nothing
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Terminal branch used to safely ignore irrelevant messages.
- **Configuration choices:**  
  Default configuration; no parameters.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**
  - Input: `If` false branch
  - Output: none
- **Version-specific requirements:**
  - Type version: `1`
- **Edge cases or potential failure types:**
  - Functionally low risk
  - May hide useful non-matching emails if the filter condition is too narrow
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
This node is intentionally used to stop processing for non-job-application emails and keep execution simple.

---

## 2.3 Data Preparation

### Overview
This block reduces the incoming Gmail payload to the fields the AI prompt needs. It standardizes the data passed to the AI agent.

### Nodes Involved
- Edit Fields

### Node Details

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Field-mapping and payload-shaping node.
- **Configuration choices:**
  - Creates three string fields:
    - `candidateEmail` from `replyTo.text`
    - `subject` from `subject`
    - `companyEmail` from `to.text`
- **Key expressions or variables used:**
  - `={{ $json.replyTo.text }}`
  - `={{ $json.subject }}`
  - `={{ $json.to.text }}`
- **Input and output connections:**
  - Input: `If` true branch
  - Output: `AI Agent`
- **Version-specific requirements:**
  - Type version: `3.4`
- **Edge cases or potential failure types:**
  - `replyTo.text` may be absent on some emails
  - `to.text` may contain display names plus email addresses rather than a raw address
  - For forwarded emails, the real applicant email may not be present in `replyTo`
  - If values are arrays or nested structures in some Gmail payloads, `.text` assumptions may not hold
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
The note explains that this block simplifies the payload for AI use. It also warns that forwarded emails may obscure the true candidate email, although in the actual implementation this node itself does not extract the email from the body; it only maps `replyTo.text`.

---

## 2.4 AI Reply Generation

### Overview
This block generates a professional HR response in a strict JSON structure. It combines an AI Agent, an OpenAI GPT-4o model, and a structured output parser to reduce formatting errors.

### Nodes Involved
- AI Agent
- OpenAI Chat Model
- Structured Output Parser

### Node Details

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based AI orchestration node that sends the prompt and attached model/parser configuration to the language model.
- **Configuration choices:**
  - Prompt type is set to “define”
  - Custom prompt positions the AI as an HR assistant at Luminar Technology
  - Instructs the model to:
    - address the applicant by name if possible
    - mention the position applied for
    - thank the applicant
    - keep the email friendly, professional, concise
    - avoid sensitive internal details
    - close with “Best regards” or “Sincerely” from the hiring team
  - Requires JSON output with:
    - `subject`
    - `body`
  - Includes input values from the current item:
    - `candidateEmail`
    - `companyEmail`
    - `subject`
  - Additional instruction: remove the candidate name from the subject
  - Output parser is enabled
- **Key expressions or variables used:**
  - `{{ $json.candidateEmail }}`
  - `{{ $json.companyEmail }}`
  - `{{ $json.subject }}`
- **Input and output connections:**
  - Main input: `Edit Fields`
  - AI language model input: `OpenAI Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Send an Email`
- **Version-specific requirements:**
  - Type version: `3.1`
  - Requires the LangChain-compatible n8n AI nodes available in the n8n instance
- **Edge cases or potential failure types:**
  - Hallucinated or malformed content if the model ignores prompt intent
  - Missing applicant name or role details because the workflow does not pass the email body into the prompt
  - Output may still fail schema validation if the model returns invalid JSON
  - Prompt interpolation may inject empty values if prior fields are missing
  - Rate limits or transient OpenAI API failures
- **Sub-workflow reference:**  
  None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model provider node for the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - No additional model options configured
  - No built-in tools enabled
- **Key expressions or variables used:**  
  None
- **Input and output connections:**
  - Output: AI language-model connection to `AI Agent`
- **Version-specific requirements:**
  - Type version: `1.3`
  - Requires valid OpenAI API credentials
  - Model availability depends on the connected OpenAI account and API access
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model not available on the account
  - OpenAI quota/rate-limit errors
  - Temporary upstream latency
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
The note states that GPT-4o is used and can be replaced if cost/performance requirements change.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a schema for AI output.
- **Configuration choices:**
  - Manual JSON schema
  - Requires:
    - `subject` as string
    - `body` as string
- **Key expressions or variables used:**  
  None
- **Input and output connections:**
  - Output: AI output-parser connection to `AI Agent`
- **Version-specific requirements:**
  - Type version: `1.3`
- **Edge cases or potential failure types:**
  - AI output that does not satisfy schema
  - If the model wraps JSON in extra commentary, parsing can fail
  - Empty strings may still technically pass schema unless further validation is added
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
The note highlights that structured output improves production stability by preventing malformed AI responses from reaching the email-sending step.

---

## 2.5 Email Delivery

### Overview
This block sends the generated reply to the candidate using SMTP. It uses the AI-generated subject and body and sends from the company email extracted earlier.

### Nodes Involved
- Send an Email

### Node Details

#### Send an Email
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  SMTP email-sending node.
- **Configuration choices:**
  - Email body:
    - `={{ $json.output.body }}`
  - Email subject:
    - `={{ $json.output.subject }}`
  - Recipient:
    - `={{ $('Edit Fields').item.json.candidateEmail }}`
  - Sender:
    - `={{ $('Edit Fields').item.json.companyEmail }}`
  - Email format:
    - `text`
  - Attribution footer disabled
- **Key expressions or variables used:**
  - `={{ $json.output.body }}`
  - `={{ $json.output.subject }}`
  - `={{ $('Edit Fields').item.json.candidateEmail }}`
  - `={{ $('Edit Fields').item.json.companyEmail }}`
- **Input and output connections:**
  - Input: `AI Agent`
  - Output: none
- **Version-specific requirements:**
  - Type version: `2.1`
  - Requires SMTP credentials configured in n8n
- **Edge cases or potential failure types:**
  - SMTP authentication or TLS/SSL misconfiguration
  - `fromEmail` may be rejected by the SMTP provider if it is not an allowed sender identity
  - `candidateEmail` may contain a display name plus address or malformed text, causing send failure
  - Plain text format may reduce readability compared with HTML
  - If AI output parser fails upstream, this node will not receive valid `output.subject` or `output.body`
- **Sub-workflow reference:**  
  None

**Sticky-note context:**  
The note recommends SMTP such as Titan Email, reminds the user to disable attribution, and suggests HTML format for better styling. However, the actual node is configured for plain text, not HTML.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 📩 Gmail Trigger – Receive Job Applications |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | This node monitors incoming emails from Gmail. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ⚠️ IMPORTANT SETUP (Forwarding Required): |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | Since this workflow is designed for custom domain emails (e.g. hello@luminartechnology.com), you must forward those emails to Gmail. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 👉 Steps: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 1. Go to your email provider (e.g. Titan Email) |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 2. Enable forwarding: hello@yourdomain.com → yourgmail@gmail.com |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ⚠️ Gmail Filtering Setup (REQUIRED): |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | Forwarded emails may: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Skip Inbox |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Be marked as read |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Not trigger n8n |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 👉 Fix this using Gmail Filters: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 1. Go to Gmail → Settings → Filters → Create Filter |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 2. Set: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | To: hello@yourdomain.com |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 3. Select: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ✔ Apply label → "job-applications" |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ✔ Mark as UNREAD |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ✔ Never send to spam |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ✔ Apply to existing messages |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 👉 In this node: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Set Label = job-applications |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | This ensures n8n correctly detects forwarded job applications. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 🤖 AI Job Application Auto-Reply System(Company Email) |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | This workflow automates HR email responses using AI. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | Flow: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 1. Receive job applications via Gmail (forwarded from domain email) |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 2. Filter relevant emails |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 3. Extract data |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 4. Generate AI-based reply |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 5. Send response automatically |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | 💡 Ideal for: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Startups |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - HR teams |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Recruitment automation |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | ⚠️ Setup Required: |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - Gmail forwarding + filtering |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - OpenAI API key |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for labeled inbound job-application emails |  | If | - SMTP configuration |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | 🔍 Filter Job Application Emails |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | This node checks if the incoming email is a job application. |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | Condition: |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | - Subject contains "Job Application" |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | 👉 You can customize this to match your hiring format: |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | Examples: |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | - "New Job Application" |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | - "Application for" |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | - "CV Submission" |
| If | n8n-nodes-base.if | Filter messages by subject content | Gmail Trigger | Edit Fields; No Operation, do nothing | Only matching emails will continue in the workflow. |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | 🧹 Prepare Email Data for AI Processing |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | This node extracts and prepares required fields: |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | - candidateEmail → extracted from email data |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | - subject → original email subject |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | - companyEmail → your receiving email |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | 👉 This simplifies the data before sending it to the AI Agent. |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | ⚠️ Note: |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | For forwarded emails, the actual candidate email may not be in the "From" field. |
| Edit Fields | n8n-nodes-base.set | Normalize input fields for AI prompt and email sending | If | AI Agent | AI will help extract the correct email from the email body. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | 🧹 Prepare Email Data for AI Processing |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | This node extracts and prepares required fields: |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | - candidateEmail → extracted from email data |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | - subject → original email subject |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | - companyEmail → your receiving email |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | 👉 This simplifies the data before sending it to the AI Agent. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | ⚠️ Note: |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | For forwarded emails, the actual candidate email may not be in the "From" field. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate a structured HR reply using LLM instructions | Edit Fields | Send an Email | AI will help extract the correct email from the email body. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | 🧠 OpenAI Model |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | This node powers the AI Agent. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | Model used: |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | - GPT-4o (can be changed) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | 👉 Requires: |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | - OpenAI API Key |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o as the LLM backend |  | AI Agent | You can switch to other models depending on cost/performance needs. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | 📦 Structured Output |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | This node ensures AI returns clean JSON output. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | Expected format: |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | { |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent |   "subject": "..." |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent |   "body": "..." |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | } |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | 👉 Why important? |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | - Prevents broken responses |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | - Ensures compatibility with Send Email node |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI output |  | AI Agent | - Makes workflow stable and production-ready |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | ✉️ Send Auto Reply to Candidate |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | This node sends the AI-generated response. |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | Configuration: |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - Uses SMTP (e.g. Titan Email) |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - Sends from your company email |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - Sends to candidate email |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | ⚙️ Requirements: |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - SMTP credentials must be configured |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - Example: |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  |   Host: smtp.titan.email |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  |   Port: 465 |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  |   Secure: SSL |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | ⚠️ Important: |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - Disable "Append Attribution" to remove n8n footer |
| Send an Email | n8n-nodes-base.emailSend | Send the generated response through SMTP | AI Agent |  | - Use HTML format for better email styling |
| No Operation, do nothing | n8n-nodes-base.noOp | Stop processing for non-matching emails | If |  | 🚫 No Operation (Ignore Non-Matching Emails) |
| No Operation, do nothing | n8n-nodes-base.noOp | Stop processing for non-matching emails | If |  | This node is used when: |
| No Operation, do nothing | n8n-nodes-base.noOp | Stop processing for non-matching emails | If |  | - Email is NOT a job application |
| No Operation, do nothing | n8n-nodes-base.noOp | Stop processing for non-matching emails | If |  | 👉 Prevents unnecessary processing |
| No Operation, do nothing | n8n-nodes-base.noOp | Stop processing for non-matching emails | If |  | Keeps workflow clean and efficient. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note on Gmail forwarding and label setup |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note on filter logic |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note on data preparation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note on AI/data-prep area |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note on OpenAI model usage |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note on structured output enforcement |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note on SMTP sending |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note on ignore branch |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | High-level documentation note for the whole workflow |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `EmailAutoReply`.
   - Keep the execution order at the default unless you have a specific reason to change it.

2. **Prepare Gmail outside n8n**
   - If your company email is on a custom domain, configure forwarding from that mailbox to a Gmail inbox.
   - In Gmail, create a filter for your company address, for example:
     - `To: hello@yourdomain.com`
   - Configure the filter to:
     - Apply a label such as `job-applications`
     - Mark as unread
     - Never send to spam
   - Ensure the label exists before configuring the Gmail Trigger in n8n.

3. **Add the `Gmail Trigger` node**
   - Node type: `Gmail Trigger`
   - Credential: create or select a **Gmail OAuth2** credential
   - Recommended setup:
     - Authenticate the Gmail account that receives the forwarded emails
   - Parameters:
     - Disable simple mode (`Simple = false`)
     - Set the filter to the Gmail label for job applications
     - Set poll frequency to every minute
   - This node will be the workflow entry point.

4. **Add the `If` node**
   - Connect `Gmail Trigger` → `If`
   - Configure a boolean condition using an expression:
     - Left value: `{{$json.subject.toLowerCase().includes("job application")}}`
     - Operator: `is true`
   - This creates:
     - True branch for matching emails
     - False branch for ignored emails

5. **Add the `No Operation, do nothing` node**
   - Node type: `No Operation`
   - Connect the **false** output of `If` to this node
   - Leave default settings

6. **Add the `Edit Fields` node**
   - Node type: `Set`
   - Connect the **true** output of `If` to `Edit Fields`
   - Add these fields as strings:
     1. `candidateEmail` = `{{$json.replyTo.text}}`
     2. `subject` = `{{$json.subject}}`
     3. `companyEmail` = `{{$json.to.text}}`
   - This node standardizes the payload for the AI step

7. **Add the `AI Agent` node**
   - Node type: `AI Agent`
   - Connect `Edit Fields` → `AI Agent`
   - Configure:
     - Prompt type: `Define`
     - Enable output parser
   - Paste the instruction prompt equivalent to:
     - The AI is an HR assistant at Luminar Technology
     - It should generate a professional reply to job applicants
     - It should thank them, mention the position, remain concise and professional, avoid internal information, and end with “Best regards” or “Sincerely”
     - It must return JSON with:
       - `subject`
       - `body`
     - Include the dynamic inputs:
       - `candidateEmail`: `{{$json.candidateEmail}}`
       - `companyEmail`: `{{$json.companyEmail}}`
       - `subject`: `{{$json.subject}}`
     - Add the rule: remove the candidate name from the subject

8. **Add the `OpenAI Chat Model` node**
   - Node type: `OpenAI Chat Model`
   - Credential: create or select an **OpenAI API** credential
   - Select model:
     - `gpt-4o`
   - Leave extra model options empty unless needed
   - Connect this node to the `AI Agent` using the **AI language model** connection, not the standard main connection

9. **Add the `Structured Output Parser` node**
   - Node type: `Structured Output Parser`
   - Configure manual schema with:
     - Object type
     - Required fields:
       - `subject` as string
       - `body` as string
   - Connect this node to the `AI Agent` using the **AI output parser** connection

10. **Add the `Send an Email` node**
    - Node type: `Send Email`
    - Connect `AI Agent` → `Send an Email`
    - Credential: create or select an **SMTP** credential
      - Typical SMTP settings depend on your provider
      - Example for Titan Email:
        - Host: `smtp.titan.email`
        - Port: `465`
        - Secure: SSL/TLS
        - Username/password: your mailbox credentials
    - Configure fields:
      - `To Email` = `{{$('Edit Fields').item.json.candidateEmail}}`
      - `From Email` = `{{$('Edit Fields').item.json.companyEmail}}`
      - `Subject` = `{{$json.output.subject}}`
      - `Text` = `{{$json.output.body}}`
      - Email format: `Text`
      - Append Attribution: disabled

11. **Verify all connections**
    - Main path:
      - `Gmail Trigger` → `If`
      - `If (true)` → `Edit Fields`
      - `Edit Fields` → `AI Agent`
      - `AI Agent` → `Send an Email`
    - Ignore path:
      - `If (false)` → `No Operation, do nothing`
    - AI side connections:
      - `OpenAI Chat Model` → `AI Agent` via language model input
      - `Structured Output Parser` → `AI Agent` via output parser input

12. **Create credentials**
    - **Gmail OAuth2**
      - Must have permission to read mailbox contents and labels
    - **OpenAI API**
      - Must have access to the selected model
    - **SMTP**
      - Must allow sending from the same company address used in `From Email`

13. **Test with a sample email**
    - Send a test application email to the company domain address
    - Confirm it is forwarded to Gmail
    - Confirm Gmail applies the target label
    - Confirm the subject contains `Job Application` so it passes the `If` node
    - Run or wait for trigger polling
    - Validate the AI output and SMTP send

14. **Harden the workflow before production**
    - Consider changing the `If` condition to avoid subject-only dependency
    - Consider extracting candidate name, job title, and email from the body explicitly
    - Consider switching the email node to HTML if you want formatted replies
    - Consider adding validation before send:
      - ensure `candidateEmail` contains a valid email address
      - ensure `output.subject` and `output.body` are non-empty
    - Consider adding error handling branches or logging

## Sub-workflow setup
This workflow does **not** use sub-workflows and does not invoke any child workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed around forwarded company emails landing in Gmail first. | Operational setup requirement |
| The filtering strategy relies on a Gmail label rather than only raw mailbox polling. | Gmail Trigger design |
| The AI prompt asks for name and role mentions, but the workflow only passes `candidateEmail`, `companyEmail`, and `subject` into the model. If the original email body is needed for better personalization, the workflow must be extended. | Important implementation limitation |
| The sticky notes recommend HTML email formatting for better presentation, but the current node is configured for plain text. | Mismatch between note and actual configuration |
| SMTP sender identity must usually match an authorized mailbox on the SMTP provider. | Email deliverability and auth consideration |
| The workflow is currently inactive (`active: false`). | Deployment status inferred from workflow metadata |