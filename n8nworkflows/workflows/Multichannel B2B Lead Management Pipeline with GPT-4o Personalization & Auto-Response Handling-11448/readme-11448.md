Multichannel B2B Lead Management Pipeline with GPT-4o Personalization & Auto-Response Handling

https://n8nworkflows.xyz/workflows/multichannel-b2b-lead-management-pipeline-with-gpt-4o-personalization---auto-response-handling-11448


# Multichannel B2B Lead Management Pipeline with GPT-4o Personalization & Auto-Response Handling

### 1. Workflow Overview

This workflow, titled **"Multichannel B2B Lead Management Pipeline with GPT-4o Personalization & Auto-Response Handling"**, automates the entire B2B outbound lead process from raw lead ingestion through AI-driven personalized outreach and reply classification to analytics and reporting. It is designed for sales development representatives (SDRs) and marketing teams aiming to scale personalized cold outreach with multichannel follow-up (email, LinkedIn, WhatsApp) and leverage AI for message generation and reply intent classification.

The workflow is logically divided into the following blocks:

- **1.1 Input & Lead Validation**: Intake of test or live leads, validating email format and checking suppression lists to filter out invalid or blocked contacts.
- **1.2 Lead Enrichment**: Calls external APIs to enrich leads with company and contact data, normalizing multiple API responses and handling enrichment failures.
- **1.3 Lead Scoring & Segmentation**: Computes a comprehensive lead score based on industry, geography, company size, revenue, pain points, and enrichment status, segmenting leads into HIGH, MEDIUM, and LOW tiers.
- **1.4 AI-Powered Outreach Generation & Sending**: Uses AI to generate personalized cold emails with subject line A/B testing, sends emails, and simulates or sends LinkedIn and WhatsApp messages with rate limiting and logging.
- **1.5 Reply Simulation & AI Classification**: Simulates lead replies or ingests real replies, classifies intent with AI into interested, follow-up, not interested, or unclear, and routes leads accordingly.
- **1.6 Lead Status Updates & Follow-up Handling**: Updates lead statuses, schedules follow-ups or meetings using AI-generated meeting drafts, and escalates unclear leads for human review with Slack notifications.
- **1.7 Analytics, Reporting & Summary**: Aggregates all results by lead score and channel, calculates statistics and cost estimates, generates AI-based human-readable summaries, logs execution metadata, and notifies the team via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Lead Validation

- **Overview:**  
  This block receives raw leads (test data or from external sources), validates email formats, checks suppression lists (blocked domains/emails), and filters out invalid or suppressed leads. Invalid leads are logged for audit.

- **Nodes Involved:**  
  Manual Trigger, Load Test Leads, Validate Email Format, Check Suppression List, Check if Invalid or Suppressed, Log Invalid Lead Event, Enrich Lead via HTTP (conditional)

- **Node Details:**  
  - *Manual Trigger*: Entry point for manual execution (test mode). No parameters.  
  - *Load Test Leads* (Code): Generates sample leads with fields like client_id, company_name, contact_email, industry, country. Outputs as JSON items.  
  - *Validate Email Format* (Code): Checks email syntax with regex, extracts domain, flags invalid emails.  
  - *Check Suppression List* (Code): Compares email/domain against hardcoded blocklists. Flags suppressed leads.  
  - *Check if Invalid or Suppressed* (If): Routes leads based on invalid or suppressed flags.  
  - *Log Invalid Lead Event* (Postgres): Logs leads that are invalid or suppressed to `lead_events` table.  
  - *Enrich Lead via HTTP* (HTTP Request): Calls external enrichment API only for valid leads.

- **Key Expressions:**  
  - Email validation regex in *Validate Email Format*.  
  - Suppression list matching in *Check Suppression List*.  
  - Conditional checks in *Check if Invalid or Suppressed*.

- **Edge Cases & Failures:**  
  - Email regex could fail on unusual but valid emails.  
  - Suppression list hardcoded; needs updating for production.  
  - Missing or malformed fields may cause downstream failures.

---

#### 1.2 Lead Enrichment

- **Overview:**  
  Enriches valid leads with additional data from one or more external APIs, normalizes data, flags enrichment success or failure, and logs events.

- **Nodes Involved:**  
  Enrich Lead via HTTP, Check Enrichment Success, Parse Enrichment Response, Merge Enrichment Data, Extra Enrichment API 2, Check Extra Enrichment Success, Normalize Enrichment Data, Log Enrichment Success, Log Enrichment Failure, Insert Enrichment Success Event, Insert Enrichment Failed Event, Set Enrichment Failed Flag

- **Node Details:**  
  - *Enrich Lead via HTTP* (HTTP Request): Calls enrichment API (example uses placeholder).  
  - *Check Enrichment Success* (If): Validates presence of `id` in response to confirm success.  
  - *Parse Enrichment Response* (Code): Merges API data with original lead, adds tech stack, employee count, revenue, pain points.  
  - *Merge Enrichment Data* (Set): Flags enrichment status as success and enriched boolean true.  
  - *Extra Enrichment API 2* (HTTP Request): Calls a second enrichment endpoint for additional data.  
  - *Check Extra Enrichment Success* (If): Similar success check.  
  - *Normalize Enrichment Data* (Code): Combines data from both APIs into unified lead record.  
  - *Log Enrichment Success/Failure* (Set + Postgres): Sets flags and logs events in DB.  
  - *Insert Enrichment Success/Failed Event* (Postgres): Inserts event records for analytics.  
  - *Set Enrichment Failed Flag* (Set): Sets default values for failed enrichments.

- **Key Expressions:**  
  - Conditional checks for enrichment success.  
  - Data extraction and merging in code nodes.

- **Edge Cases & Failures:**  
  - API timeouts or errors.  
  - Missing fields in API responses leading to incomplete enrichment.  
  - False positives in success detection if API response format changes.

---

#### 1.3 Lead Scoring & Segmentation

- **Overview:**  
  Calculates composite lead scores using weighted criteria (industry, country, company size, revenue, pain points, enrichment), segments into HIGH/MEDIUM/LOW, sets outreach tone and priority flags, and logs scoring events.

- **Nodes Involved:**  
  Compute Lead Score, Branch by Lead Score, Set HIGH Score Attributes, Set MEDIUM Score Attributes, Set LOW Score Attributes, Log HIGH Score Event, Log MEDIUM Score Event, Log LOW Score Event

- **Node Details:**  
  - *Compute Lead Score* (Code): Assigns points based on industry categories, country tiers, employee count, revenue, enrichment success, pain point length. Produces numerical score and tier string.  
  - *Branch by Lead Score* (Switch): Routes leads based on score thresholds (≥70 HIGH, ≥40 MEDIUM, else LOW).  
  - *Set HIGH/MEDIUM/LOW Score Attributes* (Set): Adds tier-specific flags like outreach tone (premium/standard/basic) and priority boolean.  
  - *Log Score Events* (Postgres): Logs lead scoring events to database.

- **Key Expressions:**  
  - Score thresholds and tier assignment logic.  
  - Scoring weightings embedded in code.

- **Edge Cases & Failures:**  
  - Missing numeric fields may skew scoring.  
  - Thresholds hardcoded, requiring adjustment for different use cases.

---

#### 1.4 AI-Powered Outreach Generation & Sending

- **Overview:**  
  Generates personalized cold emails using AI with structured output, creates subject line A/B variants, sends emails, and simulates/sends LinkedIn and WhatsApp messages with rate-limit checks and comprehensive logging.

- **Nodes Involved:**  
  AI - Generate Outreach Email, OpenAI Chat Model - Email Gen, Structured Output - Email, Extract Email Fields, Create Subject Variants A/B, Log Subject Variant, Check Email Rate Limit, Email Rate OK?, Log Email Rate Limited, Generate LinkedIn Message, Check LinkedIn Rate Limit, LinkedIn Rate OK?, Simulate LinkedIn Send, Check LinkedIn Send Success, Log LinkedIn Success, Log LinkedIn Failure, Generate WhatsApp Message, Check WhatsApp Rate Limit, WhatsApp Rate OK?, Simulate WhatsApp Send, Check WhatsApp Send Success, Log WhatsApp Success, Log WhatsApp Failure, Send Email, Prepare Email Event Log, Log Email Sent to DB

- **Node Details:**  
  - *AI - Generate Outreach Email* (LangChain Agent): Creates personalized email subject and body referencing lead data, pain points, and tech stack with a conversational tone.  
  - *OpenAI Chat Model - Email Gen*: GPT-4o-mini model called for email generation.  
  - *Structured Output - Email*: Parses AI output with schema to extract subject and body.  
  - *Extract Email Fields* (Code): Extracts generated subject and body into lead data.  
  - *Create Subject Variants A/B* (Code): Generates two subject variants for A/B testing; randomly picks one.  
  - *Log Subject Variant* (Postgres): Logs chosen variant details.  
  - *Check Email Rate Limit* (Code) and *Email Rate OK?* (If): Simulates rate limiting and quiet hours; routes accordingly.  
  - *Generate LinkedIn Message* (Code): Creates short conversational LinkedIn message based on email body.  
  - *Check LinkedIn Rate Limit* (Code) and *LinkedIn Rate OK?* (If): Checks LinkedIn message rate limits.  
  - *Simulate LinkedIn Send* (HTTP Request): Simulates sending LinkedIn message (placeholder API).  
  - *Check LinkedIn Send Success* (If): Validates send success, routes to log success or failure.  
  - *Generate WhatsApp Message* (Code): Creates brief WhatsApp/SMS message from email body with character limit enforcement.  
  - *Check WhatsApp Rate Limit* (Code) and *WhatsApp Rate OK?* (If): Rate limiting with GDPR compliance check.  
  - *Simulate WhatsApp Send* (HTTP Request): Simulates WhatsApp message sending.  
  - *Check WhatsApp Send Success* (If): Routes to log success or failure.  
  - *Send Email* (Email Send): Sends the personalized email with chosen subject and body to contact email.  
  - *Prepare Email Event Log* & *Log Email Sent to DB* (Set + Postgres): Logs email send events to DB.

- **Key Expressions:**  
  - AI prompt variables include company_name, industry, contact_name, pain_points, tech_stack.  
  - Rate limits and quiet hours logic in code nodes.  
  - Subject line A/B testing logic with random selection.

- **Edge Cases & Failures:**  
  - AI generation might produce empty or irrelevant content.  
  - Rate limit logic uses random counts; real integration requires actual counters.  
  - Email sending failures due to SMTP or credential issues.  
  - API simulation endpoints are placeholders; real integrations need replacement.

---

#### 1.5 Reply Simulation & AI Classification

- **Overview:**  
  Simulates inbound replies from leads or ingests real replies, uses AI to classify reply intent with confidence scoring and suggested next actions, and routes leads based on classified intent.

- **Nodes Involved:**  
  Simulate Reply Text, AI - Classify Reply Intent, OpenAI Chat Model - Reply Classifier, Structured Output - Classification, Route by Intent

- **Node Details:**  
  - *Simulate Reply Text* (Code): Randomly picks a reply text from lists of interested, not interested, follow-up later, or unclear responses.  
  - *AI - Classify Reply Intent* (LangChain Agent): Classifies reply intent into four categories, scores confidence, and suggests follow-up message.  
  - *OpenAI Chat Model - Reply Classifier*: GPT-4o-mini used for classification.  
  - *Structured Output - Classification*: Parses AI classification output.  
  - *Route by Intent* (Switch): Routes leads to one of four branches: interested, follow-up later, not interested, unclear.

- **Key Expressions:**  
  - AI prompt includes contact_name, company_name, and reply_text.  
  - Classification schema strictly enforces intent and confidence.

- **Edge Cases & Failures:**  
  - AI may misclassify ambiguous replies.  
  - Simulated replies may not reflect real-world distributions.  
  - Routing depends on exact string matches for intents.

---

#### 1.6 Lead Status Updates & Follow-up Handling

- **Overview:**  
  Updates lead statuses based on intent classification, schedules follow-ups and meetings using AI-generated content, escalates unclear leads for manual review with Slack alerts, and logs all status changes and notifications.

- **Nodes Involved:**  
  Mark as Qualified, AI - Draft Meeting Follow-up, OpenAI Chat Model - Meeting, Structured Output - Meeting, Create Meeting Object, Insert Meeting Record, Log Qualified Event, Check if Needs Human Review, Send Human Review Alert, Log Escalated to Human, Calculate Follow-up Date, Prepare Follow-up Event, Log Follow-up Event, Calculate Extended Follow-up Date, Insert Extended Follow-up Event, Mark as Not Interested, Log Closed Lost Event, Mark as Needs Review, Send Slack Notification, Log Manual Review Event, Log Manual Review Notification

- **Node Details:**  
  - *Mark as Qualified* (Set): Sets lead status to "qualified," adds AI confidence and suggested reply.  
  - *AI - Draft Meeting Follow-up* (LangChain Agent): Drafts meeting follow-up message with proposed time and duration.  
  - *OpenAI Chat Model - Meeting*: GPT-4o-mini model for meeting drafting.  
  - *Structured Output - Meeting*: Parses meeting draft output.  
  - *Create Meeting Object* (Code): Builds meeting record with metadata for DB insertion.  
  - *Insert Meeting Record* (Postgres): Inserts meeting into `meetings` table.  
  - *Log Qualified Event* (Postgres): Logs qualification event.  
  - *Check if Needs Human Review* (If): Determines if lead requires manual review based on intent or confidence thresholds.  
  - *Send Human Review Alert* (HTTP Request): Sends Slack alert for manual review.  
  - *Log Escalated to Human* (Postgres): Logs escalation event.  
  - *Calculate Follow-up Date* & *Calculate Extended Follow-up Date* (Code): Calculates dates 7 and 14 days ahead for follow-ups.  
  - *Prepare Follow-up Event* (Set), *Log Follow-up Event* (Postgres), *Insert Extended Follow-up Event* (Postgres): Prepare and log follow-ups.  
  - *Mark as Not Interested* (Set): Sets lead status to "closed_lost".  
  - *Log Closed Lost Event* (Postgres): Logs closed lost event.  
  - *Mark as Needs Review* (Set): Flags lead needing manual review.  
  - *Send Slack Notification* (HTTP Request): Notifies Slack channel for manual reviews.  
  - *Log Manual Review Event* and *Log Manual Review Notification* (Postgres): Logs manual review events and notifications.

- **Key Expressions:**  
  - Status flags and timestamps assigned dynamically.  
  - Meeting draft AI prompt includes contact and company context.  
  - Slack webhook URLs are placeholders and must be replaced.

- **Edge Cases & Failures:**  
  - AI meeting drafts might generate unusable content.  
  - Slack webhook failures could cause missed alerts.  
  - Date calculations assume server timezone consistency.

---

#### 1.7 Analytics, Reporting & Summary

- **Overview:**  
  Aggregates all lead data into statistics by score, channel, and status; calculates costs; generates daily analytics records; creates AI-generated human-readable summaries; logs execution metadata; and sends a run summary notification to Slack.

- **Nodes Involved:**  
  Aggregate All Results, Aggregate by Lead Score, Aggregate by Channel, Prepare Analytics Daily Row, Insert Analytics Daily Row, Calculate Final Stats, Enrich Final Stats, Estimate Lead Cost, Build Execution Metadata, Insert Execution Run Record, AI - Generate Human Summary, OpenAI Chat Model - Summary, Structured Output - Summary, Merge Summary with Stats, Prepare Final Output, Notify Team – Run Summary

- **Node Details:**  
  - *Aggregate All Results* (Aggregate): Collects all processed lead items.  
  - *Aggregate by Lead Score* (Code): Counts leads by HIGH/MEDIUM/LOW score tiers.  
  - *Aggregate by Channel* (Code): Counts leads by channel usage (email_only, LinkedIn, WhatsApp, multichannel).  
  - *Prepare Analytics Daily Row* (Code): Prepares daily summary metrics.  
  - *Insert Analytics Daily Row* (Postgres): Inserts daily summary into `analytics_daily` table.  
  - *Calculate Final Stats* (Code): Computes totals and averages for leads, enrichment, confidence, etc.  
  - *Enrich Final Stats* (Code): Adds enrichment and channel distribution metrics to stats.  
  - *Estimate Lead Cost* (Code): Estimates lead outreach costs by channel and AI calls.  
  - *Build Execution Metadata* (Code): Generates UUID run ID and aggregates key run metrics.  
  - *Insert Execution Run Record* (Postgres): Logs execution run summary.  
  - *AI - Generate Human Summary* (LangChain Agent): Summarizes workflow run in 2-3 sentences.  
  - *OpenAI Chat Model - Summary*: GPT-4o-mini for summary generation.  
  - *Structured Output - Summary*: Parses AI summary output.  
  - *Merge Summary with Stats* (Code): Combines AI summary with stats and adds final summary message.  
  - *Prepare Final Output* (Code): Builds clean output object with stats, summary, and run metadata.  
  - *Notify Team – Run Summary* (Slack): Sends formatted Slack message to #sdr-runs channel.

- **Key Expressions:**  
  - Aggregation calculations in JavaScript code nodes.  
  - AI prompt templated with key run metrics.  
  - Slack message formatting with embedded variables.

- **Edge Cases & Failures:**  
  - Aggregation depends on complete and consistent input data.  
  - AI summary may not return expected fields.  
  - Slack webhook must be valid for notifications.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                          | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                       |
|----------------------------|-----------------------------------|----------------------------------------|-------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------|
| Manual Trigger             | manualTrigger                     | Entry point for manual run              | -                                   | Load Test Leads                       | ## Input & Lead Validation Receives leads, validates email and suppression list.                 |
| Load Test Leads            | code                             | Generates test lead data                 | Manual Trigger                      | Validate Email Format                 |                                                                                                 |
| Validate Email Format      | code                             | Validates email syntax and extracts domain | Load Test Leads                   | Check Suppression List                |                                                                                                 |
| Check Suppression List     | code                             | Checks if email/domain is blocked        | Validate Email Format               | Check if Invalid or Suppressed        |                                                                                                 |
| Check if Invalid or Suppressed | if                            | Routes invalid/suppressed leads          | Check Suppression List              | Log Invalid Lead Event, Enrich Lead via HTTP |                                                                                                 |
| Log Invalid Lead Event     | postgres                         | Logs invalid or suppressed leads         | Check if Invalid or Suppressed      | Aggregate All Results                 |                                                                                                 |
| Enrich Lead via HTTP       | httpRequest                     | Calls enrichment API                      | Check if Invalid or Suppressed      | Check Enrichment Success              | ## Lead Enrichment Calls external APIs to enrich leads, normalizes and logs success/failure.     |
| Check Enrichment Success   | if                              | Checks if enrichment succeeded           | Enrich Lead via HTTP                | Parse Enrichment Response, Set Enrichment Failed Flag |                                                                                                 |
| Parse Enrichment Response  | code                            | Parses and merges enrichment data        | Check Enrichment Success            | Merge Enrichment Data                 |                                                                                                 |
| Merge Enrichment Data      | set                             | Flags enrichment success and marks enriched | Parse Enrichment Response         | Extra Enrichment API 2                |                                                                                                 |
| Extra Enrichment API 2     | httpRequest                     | Calls secondary enrichment API            | Merge Enrichment Data               | Check Extra Enrichment Success        |                                                                                                 |
| Check Extra Enrichment Success | if                          | Checks success of second enrichment       | Extra Enrichment API 2              | Normalize Enrichment Data, Log Enrichment Failure |                                                                                                 |
| Normalize Enrichment Data  | code                            | Normalizes and merges multi-API data      | Check Extra Enrichment Success      | Log Enrichment Success                |                                                                                                 |
| Log Enrichment Success     | set                             | Flags extra enrichment success            | Normalize Enrichment Data           | Insert Enrichment Success Event       |                                                                                                 |
| Insert Enrichment Success Event | postgres                   | Logs success event in DB                   | Log Enrichment Success              | Load Config                         |                                                                                                 |
| Log Enrichment Failure     | set                             | Flags extra enrichment failure            | Check Extra Enrichment Success      | Insert Enrichment Failed Event        |                                                                                                 |
| Insert Enrichment Failed Event | postgres                   | Logs failure event in DB                   | Log Enrichment Failure              | Load Config                         |                                                                                                 |
| Set Enrichment Failed Flag | set                             | Sets default fields on enrichment failure | Check Enrichment Success          | Extra Enrichment API 2                |                                                                                                 |
| Load Config                | code                            | Loads rate limits and quiet hours config   | Insert Enrichment Failed Event, Insert Enrichment Success Event | Set Geo Compliance Flags           |                                                                                                 |
| Set Geo Compliance Flags   | code                            | Flags GDPR and blocked countries          | Load Config                       | Check Do Not Contact                  |                                                                                                 |
| Check Do Not Contact       | if                              | Routes leads blocked by geo compliance    | Set Geo Compliance Flags            | Log Geo Blocked, Check GDPR Sensitive |                                                                                                 |
| Log Geo Blocked            | postgres                         | Logs geo-blocked leads                     | Check Do Not Contact                | Aggregate All Results                 |                                                                                                 |
| Check GDPR Sensitive       | if                              | Routes GDPR sensitive leads                | Check Do Not Contact                | Log GDPR Sensitive, Calculate Best Send Time |                                                                                                 |
| Log GDPR Sensitive         | postgres                         | Logs GDPR sensitive leads                  | Check GDPR Sensitive                | Calculate Best Send Time              |                                                                                                 |
| Calculate Best Send Time   | code                            | Calculates best time to send email         | Log GDPR Sensitive                  | Log Send Time Planned                 |                                                                                                 |
| Log Send Time Planned      | postgres                         | Logs planned send time                      | Calculate Best Send Time            | Compute Lead Score                   |                                                                                                 |
| Compute Lead Score         | code                            | Calculates composite lead score             | Log Send Time Planned               | Branch by Lead Score                  | ## Lead Scoring & Segmentation Calculates lead scores and segments leads into tiers.            |
| Branch by Lead Score       | switch                          | Routes leads by score tier                  | Compute Lead Score                  | Set HIGH/MEDIUM/LOW Score Attributes |                                                                                                 |
| Set HIGH Score Attributes  | set                             | Sets flags for HIGH score leads             | Branch by Lead Score                | Log HIGH Score Event                  |                                                                                                 |
| Log HIGH Score Event       | postgres                         | Logs HIGH score event                       | Set HIGH Score Attributes           | AI - Generate Outreach Email          |                                                                                                 |
| Set MEDIUM Score Attributes | set                            | Sets flags for MEDIUM score leads           | Branch by Lead Score                | Log MEDIUM Score Event                |                                                                                                 |
| Log MEDIUM Score Event     | postgres                         | Logs MEDIUM score event                     | Set MEDIUM Score Attributes         | AI - Generate Outreach Email          |                                                                                                 |
| Set LOW Score Attributes   | set                             | Sets flags for LOW score leads              | Branch by Lead Score                | Log LOW Score Event                   |                                                                                                 |
| Log LOW Score Event        | postgres                         | Logs LOW score event                        | Set LOW Score Attributes            | AI - Generate Outreach Email          |                                                                                                 |
| AI - Generate Outreach Email | langchain.agent               | Generates personalized cold email           | Log HIGH/MEDIUM/LOW Score Event    | Extract Email Fields                  | ## AI Email Generation & Send Uses AI to generate personalized cold emails with A/B testing.    |
| OpenAI Chat Model - Email Gen | langchain.lmChatOpenAi       | Calls GPT-4o-mini model for email gen       | AI - Generate Outreach Email       | Structured Output - Email             |                                                                                                 |
| Structured Output - Email  | langchain.outputParserStructured | Parses AI output into subject and body      | OpenAI Chat Model - Email Gen      | Extract Email Fields                  |                                                                                                 |
| Extract Email Fields       | code                            | Extracts subject and body from AI output    | AI - Generate Outreach Email       | Create Subject Variants A/B           |                                                                                                 |
| Create Subject Variants A/B | code                           | Creates subject line variants for A/B testing | Extract Email Fields             | Log Subject Variant                   |                                                                                                 |
| Log Subject Variant        | postgres                         | Logs chosen subject variant                  | Create Subject Variants A/B         | Check Email Rate Limit                |                                                                                                 |
| Check Email Rate Limit     | code                            | Checks if email sending is within limits     | Log Subject Variant                | Email Rate OK?                       |                                                                                                 |
| Email Rate OK?             | if                              | Routes based on email rate limit              | Check Email Rate Limit             | Log Email Rate Limited, Generate LinkedIn Message |                                                                                                 |
| Log Email Rate Limited     | postgres                         | Logs when email rate limit is exceeded        | Email Rate OK?                    | Generate LinkedIn Message             |                                                                                                 |
| Generate LinkedIn Message  | code                            | Creates LinkedIn DM message from email body  | Email Rate OK?                    | Check LinkedIn Rate Limit             |                                                                                                 |
| Check LinkedIn Rate Limit  | code                            | Checks LinkedIn message send limits           | Generate LinkedIn Message          | LinkedIn Rate OK?                    |                                                                                                 |
| LinkedIn Rate OK?          | if                              | Routes based on LinkedIn rate limit            | Check LinkedIn Rate Limit          | Log LinkedIn Rate Limited, Simulate LinkedIn Send |                                                                                                 |
| Log LinkedIn Rate Limited  | postgres                         | Logs LinkedIn rate limiting                    | LinkedIn Rate OK?                 | Check LinkedIn Send Success           |                                                                                                 |
| Simulate LinkedIn Send     | httpRequest                     | Simulates LinkedIn message sending             | LinkedIn Rate OK?                 | Check LinkedIn Send Success           |                                                                                                 |
| Check LinkedIn Send Success | if                             | Checks success of LinkedIn send                 | Simulate LinkedIn Send            | Log LinkedIn Success, Log LinkedIn Failure |                                                                                                 |
| Log LinkedIn Success       | postgres                         | Logs LinkedIn send success                      | Check LinkedIn Send Success       | Simulate WhatsApp Send                |                                                                                                 |
| Log LinkedIn Failure       | postgres                         | Logs LinkedIn send failure                      | Check LinkedIn Send Success       | Simulate WhatsApp Send                |                                                                                                 |
| Generate WhatsApp Message  | code                            | Creates WhatsApp/SMS message from email body   | Log LinkedIn Success, Log LinkedIn Failure | Check WhatsApp Rate Limit       |                                                                                                 |
| Check WhatsApp Rate Limit  | code                            | Checks WhatsApp message sending limits          | Generate WhatsApp Message         | WhatsApp Rate OK?                    |                                                                                                 |
| WhatsApp Rate OK?          | if                              | Routes based on WhatsApp rate limit               | Check WhatsApp Rate Limit         | Log WhatsApp Rate Limited, Simulate WhatsApp Send |                                                                                                 |
| Log WhatsApp Rate Limited  | postgres                         | Logs WhatsApp rate limiting                      | WhatsApp Rate OK?                 | Check WhatsApp Send Success           |                                                                                                 |
| Simulate WhatsApp Send     | httpRequest                     | Simulates WhatsApp message sending               | WhatsApp Rate OK?                 | Check WhatsApp Send Success, Check WhatsApp Rate Limit |                                                                                                 |
| Check WhatsApp Send Success | if                             | Checks WhatsApp send success                      | Simulate WhatsApp Send            | Log WhatsApp Success, Log WhatsApp Failure |                                                                                                 |
| Log WhatsApp Success       | postgres                         | Logs WhatsApp send success                        | Check WhatsApp Send Success       | Send Email                          |                                                                                                 |
| Log WhatsApp Failure       | postgres                         | Logs WhatsApp send failure                        | Check WhatsApp Send Success       | Send Email                          |                                                                                                 |
| Send Email                 | emailSend                       | Sends generated email to lead                      | Check if Multichannel Eligible     | Prepare Email Event Log               |                                                                                                 |
| Prepare Email Event Log    | set                             | Prepares event data for email sent logging         | Send Email                      | Log Email Sent to DB                  |                                                                                                 |
| Log Email Sent to DB       | postgres                         | Logs email sent event                             | Prepare Email Event Log           | Simulate Reply Text                   |                                                                                                 |
| Simulate Reply Text        | code                            | Simulates different types of lead replies          | Log Email Sent to DB              | AI - Classify Reply Intent            | ## Replies & AI Classification Simulates or ingests replies and classifies intent with AI.      |
| AI - Classify Reply Intent | langchain.agent                 | Classifies reply intent and confidence               | Simulate Reply Text              | Route by Intent                      |                                                                                                 |
| OpenAI Chat Model - Reply Classifier | langchain.lmChatOpenAi   | GPT-4o-mini model for reply classification           | AI - Classify Reply Intent       | Structured Output - Classification    |                                                                                                 |
| Structured Output - Classification | langchain.outputParserStructured | Parses AI reply classification output             | OpenAI Chat Model - Reply Classifier | Route by Intent                    |                                                                                                 |
| Route by Intent            | switch                          | Routes leads by classified intent                    | AI - Classify Reply Intent       | Mark as Qualified, Calculate Follow-up Date, Mark as Not Interested, Mark as Needs Review |                                                                                                 |
| Mark as Qualified          | set                             | Updates lead status to "qualified"                    | Route by Intent (interested)     | AI - Draft Meeting Follow-up          | ## Status & Follow-up Updates statuses, schedules meetings/follow-ups, escalates for review.    |
| AI - Draft Meeting Follow-up | langchain.agent               | Generates meeting follow-up message with time         | Mark as Qualified               | Create Meeting Object                |                                                                                                 |
| OpenAI Chat Model - Meeting | langchain.lmChatOpenAi         | GPT-4o-mini for meeting message generation            | AI - Draft Meeting Follow-up    | Structured Output - Meeting           |                                                                                                 |
| Structured Output - Meeting | langchain.outputParserStructured | Parses meeting draft output                           | OpenAI Chat Model - Meeting      | Create Meeting Object                |                                                                                                 |
| Create Meeting Object      | code                            | Creates meeting record object for DB insertion         | AI - Draft Meeting Follow-up    | Insert Meeting Record                |                                                                                                 |
| Insert Meeting Record      | postgres                         | Inserts meeting record in DB                            | Create Meeting Object            | Log Qualified Event                 |                                                                                                 |
| Log Qualified Event        | postgres                         | Logs lead qualified event                               | Insert Meeting Record            | Aggregate All Results               |                                                                                                 |
| Check if Needs Human Review | if                             | Checks if lead needs manual review                      | Route by Intent (interested)     | Send Human Review Alert, Mark as Qualified |                                                                                                 |
| Send Human Review Alert    | httpRequest                     | Sends Slack alert for manual review                      | Check if Needs Human Review      | Log Escalated to Human              |                                                                                                 |
| Log Escalated to Human     | postgres                         | Logs escalation to human event                           | Send Human Review Alert          | Mark as Qualified                  |                                                                                                 |
| Calculate Follow-up Date   | code                            | Calculates follow-up date 7 days ahead                   | Route by Intent (follow_up_later) | Prepare Follow-up Event            |                                                                                                 |
| Prepare Follow-up Event    | set                             | Prepares follow-up event data                             | Calculate Follow-up Date         | Log Follow-up Event                |                                                                                                 |
| Log Follow-up Event        | postgres                         | Logs scheduled follow-up event                            | Prepare Follow-up Event          | Aggregate All Results              |                                                                                                 |
| Calculate Extended Follow-up Date | code                      | Calculates extended follow-up date 14 days ahead          | Calculate Follow-up Date         | Insert Extended Follow-up Event     |                                                                                                 |
| Insert Extended Follow-up Event | postgres                    | Logs extended follow-up event                             | Calculate Extended Follow-up Date | Prepare Follow-up Event           |                                                                                                 |
| Mark as Not Interested     | set                             | Marks lead as closed lost                                 | Route by Intent (not_interested) | Log Closed Lost Event              |                                                                                                 |
| Log Closed Lost Event      | postgres                         | Logs closed lost event                                    | Mark as Not Interested           | Aggregate All Results              |                                                                                                 |
| Mark as Needs Review       | set                             | Flags lead for manual review                              | Route by Intent (unclear)        | Send Slack Notification           |                                                                                                 |
| Send Slack Notification    | httpRequest                     | Sends Slack notification for manual review                | Mark as Needs Review             | Log Manual Review Notification    |                                                                                                 |
| Log Manual Review Event    | postgres                         | Logs manual review event                                  | Send Slack Notification          | Aggregate All Results              |                                                                                                 |
| Log Manual Review Notification | postgres                    | Logs Slack notification event                             | Send Slack Notification          | -                                 |                                                                                                 |
| Aggregate All Results      | aggregate                       | Aggregates all processed lead data                        | Log Invalid Lead Event, Log Qualified Event, Log Follow-up Event, Log Closed Lost Event, Log Manual Review Event, etc. | Aggregate by Lead Score            | ## Analytics & Reporting Aggregates results, calculates metrics and costs, generates summaries. |
| Aggregate by Lead Score    | code                            | Counts leads by score tier                                 | Aggregate All Results            | Aggregate by Channel              |                                                                                                 |
| Aggregate by Channel       | code                            | Counts leads by outreach channel usage                     | Aggregate by Lead Score          | Prepare Analytics Daily Row       |                                                                                                 |
| Prepare Analytics Daily Row | code                           | Prepares daily analytics summary row                       | Aggregate by Channel             | Insert Analytics Daily Row        |                                                                                                 |
| Insert Analytics Daily Row | postgres                         | Inserts daily analytics record                             | Prepare Analytics Daily Row      | Enrich Final Stats, Estimate Lead Cost |                                                                                                 |
| Calculate Final Stats      | code                            | Calculates overall stats and averages                      | Aggregate All Results            | Build Execution Metadata          |                                                                                                 |
| Enrich Final Stats         | code                            | Adds enrichment & channel distributions to stats          | Insert Analytics Daily Row       | Estimate Lead Cost                |                                                                                                 |
| Estimate Lead Cost         | code                            | Estimates cost for outreach channels and AI calls          | Insert Analytics Daily Row       | Enrich Final Stats               |                                                                                                 |
| Build Execution Metadata   | code                            | Builds run metadata with UUID, timestamp, and metrics      | Calculate Final Stats            | Insert Execution Run Record       |                                                                                                 |
| Insert Execution Run Record | postgres                       | Inserts execution run record                               | Build Execution Metadata         | AI - Generate Human Summary       |                                                                                                 |
| AI - Generate Human Summary | langchain.agent               | Generates human-readable summary of workflow run           | Insert Execution Run Record      | Merge Summary with Stats          |                                                                                                 |
| OpenAI Chat Model - Summary | langchain.lmChatOpenAi         | GPT-4o-mini model for summary generation                    | AI - Generate Human Summary      | Structured Output - Summary       |                                                                                                 |
| Structured Output - Summary | langchain.outputParserStructured | Parses AI summary output                                  | OpenAI Chat Model - Summary      | Merge Summary with Stats          |                                                                                                 |
| Merge Summary with Stats   | code                            | Merges AI summary with stats                                | AI - Generate Human Summary, Calculate Final Stats | Prepare Final Output        |                                                                                                 |
| Prepare Final Output       | code                            | Prepares clean final output with run metadata and stats    | Merge Summary with Stats         | Notify Team – Run Summary          |                                                                                                 |
| Notify Team – Run Summary  | slack                           | Sends Slack notification with run summary                   | Prepare Final Output             | -                                | ## Analytics & Reporting Sends final summary to Slack channel (#sdr-runs).                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing.

2. **Create Load Test Leads Node (Code)**  
   - Type: Code  
   - Code: Define sample leads JSON array with fields: client_id, company_name, website, contact_name, contact_email, industry, country.  
   - Connect from Manual Trigger.

3. **Validate Email Format Node (Code)**  
   - Validate email syntax using regex.  
   - Extract domain. Add flags `invalid_email` and `email_domain`.  
   - Connect from Load Test Leads.

4. **Check Suppression List Node (Code)**  
   - Define arrays for blocked domains and emails.  
   - Flag leads as blocked if match found. Add `is_suppressed`.  
   - Connect from Validate Email Format.

5. **Check if Invalid or Suppressed Node (If)**  
   - Condition: `invalid_email` OR `is_suppressed` equals true.  
   - True branch: Log Invalid Lead Event.  
   - False branch: Enrich Lead via HTTP.

6. **Log Invalid Lead Event Node (Postgres)**  
   - Table: `lead_events`  
   - Log client_id, lead_email, event_type = "suppressed", payload with reason.  
   - Connect True output from Check if Invalid or Suppressed.

7. **Enrich Lead via HTTP Node (HTTP Request)**  
   - URL: External enrichment API (replace placeholder).  
   - Response format: JSON.  
   - Connect False output from Check if Invalid or Suppressed.

8. **Check Enrichment Success Node (If)**  
   - Condition: Response JSON field `id` is not empty.  
   - True: Parse Enrichment Response.  
   - False: Set Enrichment Failed Flag.

9. **Parse Enrichment Response Node (Code)**  
   - Merge enrichment API data with original lead data.  
   - Add fields like company name, tech stack, employee count, estimated revenue, pain points.  
   - Connect from Check Enrichment Success (True).

10. **Merge Enrichment Data Node (Set)**  
    - Set `enrichment_status` to "success" and `enriched` to true.  
    - Connect from Parse Enrichment Response.

11. **Extra Enrichment API 2 Node (HTTP Request)**  
    - Secondary enrichment API call (placeholder URL).  
    - Connect from Merge Enrichment Data.

12. **Check Extra Enrichment Success Node (If)**  
    - Condition: Response JSON field `id` is not empty.  
    - True: Normalize Enrichment Data.  
    - False: Log Enrichment Failure.

13. **Normalize Enrichment Data Node (Code)**  
    - Combine first and second enrichment API data into unified lead object.  
    - Connect from Check Extra Enrichment Success (True).

14. **Log Enrichment Success Node (Set)**  
    - Set `extra_enrichment_status` to "success".  
    - Connect from Normalize Enrichment Data.

15. **Insert Enrichment Success Event Node (Postgres)**  
    - Log enrichment success event to `lead_events` table.  
    - Connect from Log Enrichment Success.

16. **Log Enrichment Failure Node (Set)**  
    - Set `extra_enrichment_status` to "failed".  
    - Connect from Check Extra Enrichment Success (False).

17. **Insert Enrichment Failed Event Node (Postgres)**  
    - Log enrichment failure event to `lead_events`.  
    - Connect from Log Enrichment Failure.

18. **Set Enrichment Failed Flag Node (Set)**  
    - Set `enrichment_status` to "failed", `enriched` false, default empty values for tech stack, employee count, etc.  
    - Connect from Check Enrichment Success (False).

19. **Load Config Node (Code)**  
    - Define rate limits (email, LinkedIn, WhatsApp), quiet hours config.  
    - Calculate if current time is quiet hours.  
    - Connect from Insert Enrichment Success/Failed Event.

20. **Set Geo Compliance Flags Node (Code)**  
    - Flag GDPR sensitive and blocked countries based on country field.  
    - Connect from Load Config.

21. **Check Do Not Contact Node (If)**  
    - Condition: `do_not_contact` flag true.  
    - True: Log Geo Blocked.  
    - False: Check GDPR Sensitive.

22. **Log Geo Blocked Node (Postgres)**  
    - Log geo blocked event.  
    - Connect from Check Do Not Contact (True).

23. **Check GDPR Sensitive Node (If)**  
    - Condition: `gdpr_sensitive` true.  
    - True: Log GDPR Sensitive.  
    - False: Calculate Best Send Time.

24. **Log GDPR Sensitive Node (Postgres)**  
    - Log GDPR sensitive event.  
    - Connect from Check GDPR Sensitive (True).

25. **Calculate Best Send Time Node (Code)**  
    - Set timezone by country, set best send time to next day 10AM local time.  
    - Connect from Check GDPR Sensitive (False).

26. **Log Send Time Planned Node (Postgres)**  
    - Log planned send time for email.  
    - Connect from Calculate Best Send Time.

27. **Compute Lead Score Node (Code)**  
    - Score leads by industry, country, employee count, revenue, enrichment success, pain points length.  
    - Output score and tier (HIGH, MEDIUM, LOW).  
    - Connect from Log Send Time Planned.

28. **Branch by Lead Score Node (Switch)**  
    - Route leads into HIGH (≥70), MEDIUM (≥40), LOW (<40).  
    - Connect from Compute Lead Score.

29. **Set HIGH Score Attributes Node (Set)**  
    - Set outreach tone "premium", priority_flag true, score_tier "HIGH".  
    - Connect from Branch by Lead Score (HIGH).

30. **Log HIGH Score Event Node (Postgres)**  
    - Log lead scored high.  
    - Connect from Set HIGH Score Attributes.

31. **Set MEDIUM Score Attributes Node (Set)**  
    - Set outreach tone "standard", priority_flag false, score_tier "MEDIUM".  
    - Connect from Branch by Lead Score (MEDIUM).

32. **Log MEDIUM Score Event Node (Postgres)**  
    - Log lead scored medium.  
    - Connect from Set MEDIUM Score Attributes.

33. **Set LOW Score Attributes Node (Set)**  
    - Set outreach tone "basic", priority_flag false, score_tier "LOW".  
    - Connect from Branch by Lead Score (LOW).

34. **Log LOW Score Event Node (Postgres)**  
    - Log lead scored low.  
    - Connect from Set LOW Score Attributes.

35. **AI - Generate Outreach Email Node (LangChain Agent)**  
    - Prompt AI to write personalized cold outreach email based on lead info and pain points.  
    - Connect from all Log Score Event nodes.

36. **OpenAI Chat Model - Email Gen Node**  
    - GPT-4o-mini model call.  
    - Connect from AI - Generate Outreach Email.

37. **Structured Output - Email Node**  
    - Parse AI output into subject and body fields.  
    - Connect from OpenAI Chat Model - Email Gen.

38. **Extract Email Fields Node (Code)**  
    - Extract subject and body from AI output, merge to lead data.  
    - Connect from AI - Generate Outreach Email.

39. **Create Subject Variants A/B Node (Code)**  
    - Generate two subject variants, randomly select one as final.  
    - Connect from Extract Email Fields.

40. **Log Subject Variant Node (Postgres)**  
    - Log chosen subject variant.  
    - Connect from Create Subject Variants A/B.

41. **Check Email Rate Limit Node (Code)**  
    - Check if sending email is within rate limit and not quiet hours.  
    - Connect from Log Subject Variant.

42. **Email Rate OK? Node (If)**  
    - True: Generate LinkedIn Message.  
    - False: Log Email Rate Limited.  
    - Connect from Check Email Rate Limit.

43. **Log Email Rate Limited Node (Postgres)**  
    - Log email rate limiting event.  
    - Connect from Email Rate OK? (False).

44. **Generate LinkedIn Message Node (Code)**  
    - Generate short LinkedIn DM based on email body.  
    - Connect from Email Rate OK? (True).

45. **Check LinkedIn Rate Limit Node (Code)**  
    - Check LinkedIn message send counts within limits.  
    - Connect from Generate LinkedIn Message.

46. **LinkedIn Rate OK? Node (If)**  
    - True: Simulate LinkedIn Send.  
    - False: Log LinkedIn Rate Limited.  
    - Connect from Check LinkedIn Rate Limit.

47. **Log LinkedIn Rate Limited Node (Postgres)**  
    - Log LinkedIn rate limiting event.  
    - Connect from LinkedIn Rate OK? (False).

48. **Simulate LinkedIn Send Node (HTTP Request)**  
    - Simulate sending LinkedIn message.  
    - Connect from LinkedIn Rate OK? (True).

49. **Check LinkedIn Send Success Node (If)**  
    - True: Log LinkedIn Success.  
    - False: Log LinkedIn Failure.  
    - Connect from Simulate LinkedIn Send.

50. **Log LinkedIn Success Node (Postgres)**  
    - Log successful LinkedIn send.  
    - Connect from Check LinkedIn Send Success (True).

51. **Log LinkedIn Failure Node (Postgres)**  
    - Log failed LinkedIn send.  
    - Connect from Check LinkedIn Send Success (False).

52. **Generate WhatsApp Message Node (Code)**  
    - Generate brief WhatsApp/SMS message from email body, enforce 160 char limit.  
    - Connect from Log LinkedIn Success and Log LinkedIn Failure.

53. **Check WhatsApp Rate Limit Node (Code)**  
    - Check WhatsApp send counts and GDPR compliance.  
    - Connect from Generate WhatsApp Message.

54. **WhatsApp Rate OK? Node (If)**  
    - True: Simulate WhatsApp Send.  
    - False: Log WhatsApp Rate Limited.  
    - Connect from Check WhatsApp Rate Limit.

55. **Log WhatsApp Rate Limited Node (Postgres)**  
    - Log WhatsApp rate limiting event.  
    - Connect from WhatsApp Rate OK? (False).

56. **Simulate WhatsApp Send Node (HTTP Request)**  
    - Simulate WhatsApp message sending.  
    - Connect from WhatsApp Rate OK? (True).

57. **Check WhatsApp Send Success Node (If)**  
    - True: Log WhatsApp Success.  
    - False: Log WhatsApp Failure.  
    - Connect from Simulate WhatsApp Send.

58. **Log WhatsApp Success Node (Postgres)**  
    - Log successful WhatsApp send.  
    - Connect from Check WhatsApp Send Success (True).

59. **Log WhatsApp Failure Node (Postgres)**  
    - Log failed WhatsApp send.  
    - Connect from Check WhatsApp Send Success (False).

60. **Send Email Node (Email Send)**  
    - Sends personalized cold email to lead.  
    - Connect from Check if Multichannel Eligible (conditionally).  

61. **Prepare Email Event Log Node (Set)**  
    - Prepares event data for email sent logging.  
    - Connect from Send Email.

62. **Log Email Sent to DB Node (Postgres)**  
    - Logs email sent event.  
    - Connect from Prepare Email Event Log.

63. **Simulate Reply Text Node (Code)**  
    - Simulates lead replies from predefined lists.  
    - Connect from Log Email Sent to DB.

64. **AI - Classify Reply Intent Node (LangChain Agent)**  
    - Classifies reply intent, confidence, and suggests follow-up.  
    - Connect from Simulate Reply Text.

65. **OpenAI Chat Model - Reply Classifier Node**  
    - GPT-4o-mini model for reply classification.  
    - Connect from AI - Classify Reply Intent.

66. **Structured Output - Classification Node**  
    - Parses AI classification output.  
    - Connect from OpenAI Chat Model - Reply Classifier.

67. **Route by Intent Node (Switch)**  
    - Routes leads by intent: interested, follow_up_later, not_interested, unclear.  
    - Connect from AI - Classify Reply Intent.

68. **Mark as Qualified Node (Set)**  
    - Sets lead status "qualified", adds AI confidence, suggested reply.  
    - Connect from Route by Intent (interested).

69. **AI - Draft Meeting Follow-up Node (LangChain Agent)**  
    - Drafts follow-up meeting message with time suggestion.  
    - Connect from Mark as Qualified.

70. **OpenAI Chat Model - Meeting Node**  
    - GPT-4o-mini model for meeting drafts.  
    - Connect from AI - Draft Meeting Follow-up.

71. **Structured Output - Meeting Node**  
    - Parses meeting draft output.  
    - Connect from OpenAI Chat Model - Meeting.

72. **Create Meeting Object Node (Code)**  
    - Creates meeting object with metadata for DB.  
    - Connect from AI - Draft Meeting Follow-up.

73. **Insert Meeting Record Node (Postgres)**  
    - Inserts meeting record to `meetings` table.  
    - Connect from Create Meeting Object.

74. **Log Qualified Event Node (Postgres)**  
    - Logs lead qualified event.  
    - Connect from Insert Meeting Record.

75. **Check if Needs Human Review Node (If)**  
    - Checks if lead requires manual review based on intent/confidence.  
    - True: Send Human Review Alert.  
    - False: Mark as Qualified.  
    - Connect from Route by Intent (interested).

76. **Send Human Review Alert Node (HTTP Request)**  
    - Sends Slack alert for manual review.  
    - Connect from Check if Needs Human Review (True).

77. **Log Escalated to Human Node (Postgres)**  
    - Logs escalation event.  
    - Connect from Send Human Review Alert.

78. **Calculate Follow-up Date Node (Code)**  
    - Calculates follow-up date 7 days ahead.  
    - Connect from Route by Intent (follow_up_later).

79. **Prepare Follow-up Event Node (Set)**  
    - Prepares follow-up data with status and scheduled date.  
    - Connect from Calculate Follow-up Date.

80. **Log Follow-up Event Node (Postgres)**  
    - Logs follow-up scheduling event.  
    - Connect from Prepare Follow-up Event.

81. **Calculate Extended Follow-up Date Node (Code)**  
    - Calculates follow-up date 14 days ahead.  
    - Connect from Calculate Follow-up Date.

82. **Insert Extended Follow-up Event Node (Postgres)**  
    - Logs extended follow-up event.  
    - Connect from Calculate Extended Follow-up Date.

83. **Mark as Not Interested Node (Set)**  
    - Sets lead status "closed_lost", records closed date.  
    - Connect from Route by Intent (not_interested).

84. **Log Closed Lost Event Node (Postgres)**  
    - Logs closed lost event.  
    - Connect from Mark as Not Interested.

85. **Mark as Needs Review Node (Set)**  
    - Flags lead needing manual review.  
    - Connect from Route by Intent (unclear).

86. **Send Slack Notification Node (HTTP Request)**  
    - Sends Slack notification for manual review.  
    - Connect from Mark as Needs Review.

87. **Log Manual Review Event Node (Postgres)**  
    - Logs manual review event.  
    - Connect from Send Slack Notification.

88. **Log Manual Review Notification Node (Postgres)**  
    - Logs Slack notification event.  
    - Connect from Send Slack Notification.

89. **Aggregate All Results Node (Aggregate)**  
    - Aggregates all processed lead data items.  
    - Connect from multiple event log nodes.

90. **Aggregate by Lead Score Node (Code)**  
    - Counts leads by score tier (HIGH, MEDIUM, LOW).  
    - Connect from Aggregate All Results.

91. **Aggregate by Channel Node (Code)**  
    - Counts leads by channel used (email_only, LinkedIn, WhatsApp, multichannel).  
    - Connect from Aggregate by Lead Score.

92. **Prepare Analytics Daily Row Node (Code)**  
    - Prepares daily analytics summary metrics.  
    - Connect from Aggregate by Channel.

93. **Insert Analytics Daily Row Node (Postgres)**  
    - Inserts daily analytics row into `analytics_daily` table.  
    - Connect from Prepare Analytics Daily Row.

94. **Calculate Final Stats Node (Code)**  
    - Calculates final summary statistics (counts, rates, averages).  
    - Connect from Aggregate All Results.

95. **Enrich Final Stats Node (Code)**  
    - Adds enrichment, channel, and score distributions to stats.  
    - Connect from Insert Analytics Daily Row.

96. **Estimate Lead Cost Node (Code)**  
    - Calculates estimated outreach cost by channel and AI calls.  
    - Connect from Insert Analytics Daily Row.

97. **Build Execution Metadata Node (Code)**  
    - Generates run UUID and compiles execution metadata.  
    - Connect from Calculate Final Stats.

98. **Insert Execution Run Record Node (Postgres)**  
    - Inserts execution run metadata record.  
    - Connect from Build Execution Metadata.

99. **AI - Generate Human Summary Node (LangChain Agent)**  
    - Generates a 2-3 sentence human-readable summary of run metrics.  
    - Connect from Insert Execution Run Record.

100. **OpenAI Chat Model - Summary Node (LangChain)**  
    - GPT-4o-mini model for run summary generation.  
    - Connect from AI - Generate Human Summary.

101. **Structured Output - Summary Node (LangChain)**  
    - Parses AI-generated summary output.  
    - Connect from OpenAI Chat Model - Summary.

102. **Merge Summary with Stats Node (Code)**  
    - Combines AI summary with stats and adds final message.  
    - Connect from AI - Generate Human Summary and Calculate Final Stats.

103. **Prepare Final Output Node (Code)**  
    - Formats final output object with run ID, timestamp, stats, and summary.  
    - Connect from Merge Summary with Stats.

104. **Notify Team – Run Summary Node (Slack)**  
    - Sends Slack notification with run summary to #sdr-runs channel.  
    - Connect from Prepare Final Output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow takes raw B2B leads, validates and enriches them, then uses AI to send personalized cold emails and classify replies.                                                                                                     | Sticky Note at workflow start                                                                   |
| Setup steps include connecting credentials (email, enrichment API, database, Slack), reviewing suppression rules, editing AI prompts, setting real data sources, and testing with small lists before going live.                           | Sticky Note                                                                                     |
| Input & Lead Validation block filters leads by email validity and suppression lists before enrichment.                                                                                                                                   | Sticky Note near input nodes                                                                    |
| Lead Enrichment calls multiple APIs, normalizes data, and handles failures robustly.                                                                                                                                                      | Sticky Note near enrichment nodes                                                              |
| Lead Scoring & Segmentation computes scores from multiple signals and segments leads for tailored outreach.                                                                                                                             | Sticky Note near scoring nodes                                                                 |
| AI Email Generation & Send handles personalized email creation, A/B subject testing, multi-channel messaging, and logs all sends.                                                                                                        | Sticky Note near AI email and sending nodes                                                   |
| Replies & AI Classification simulate or ingest replies, classify intent, and route leads accordingly.                                                                                                                                   | Sticky Note near reply processing nodes                                                       |
| Status & Follow-up updates lead statuses, schedules follow-ups or meetings, escalates leads for manual review with Slack notifications.                                                                                                | Sticky Note near status update nodes                                                          |
| Analytics & Reporting aggregates and analyzes results, estimates costs, generates AI summaries, and notifies teams via Slack.                                                                                                           | Sticky Note near analytics and reporting nodes                                               |
| Slack webhooks and API URLs used are placeholders and should be replaced with your actual credentials and endpoints.                                                                                                                    | Throughout HTTP Request nodes                                                                  |
| AI nodes use GPT-4o-mini model; ensure you have OpenAI credentials configured in n8n.                                                                                                                                                    | AI LangChain nodes                                                                            |
| Rate limiting nodes simulate counts with random values; replace with real counters or API checks in production.                                                                                                                        | Rate limit checking code nodes                                                                |

---

**Disclaimer:**  
The text above is exclusively derived from an n8n automation workflow. It strictly complies with content policies and contains no illegal, offensive, or protected content. All processed data is legal and publicly accessible.