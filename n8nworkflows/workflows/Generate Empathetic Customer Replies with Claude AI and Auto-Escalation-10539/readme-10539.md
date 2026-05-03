Generate Empathetic Customer Replies with Claude AI and Auto-Escalation

https://n8nworkflows.xyz/workflows/generate-empathetic-customer-replies-with-claude-ai-and-auto-escalation-10539


# Generate Empathetic Customer Replies with Claude AI and Auto-Escalation

### 1. Workflow Overview

This n8n workflow automates the generation of empathetic customer support replies using Claude AI (Anthropic) and routes messages for auto-escalation based on risk detection and confidence thresholds. It targets customer-facing teams such as Support, Community, Sales, and CX, enabling quick, sensitive, and safe response drafts that either can be sent automatically or flagged for human review.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives incoming messages via manual trigger, webhook (real-time), or scheduled polling.
- **1.2 Configuration Setup**: Defines parameters controlling reply length, tone, risk keywords, and sanitization rules.
- **1.3 AI Processing**: Sends input text to an AI agent built with Claude (Anthropic) to analyze sentiment, tone, and generate a draft reply.
- **1.4 Output Parsing and Sanitization**: Cleans the generated reply by removing links, emojis, personal identifiable information (PII), and truncates to max length.
- **1.5 Risk Assessment & Escalation Decision**: Applies keyword matching, confidence scoring, and sentiment checks to decide if the reply needs human handover.
- **1.6 Routing & Final Output**: Routes the message for either auto-approval or review and marks the message status accordingly.

Supporting nodes include logging, Slack integration, Google Sheets logging, and sticky notes with documentation and usage instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Handles message input via manual testing, webhook for real-time messages, or scheduled polling for missed/queued messages.

**Nodes Involved:**  
- `for testing` (Manual Trigger)  
- `for real-time replies` (Webhook)  
- `(5–10 min) for missed or queued` (Schedule Trigger)  
- `Test Input (Change Me)1` (Set Node)

**Node Details:**

- **for testing**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing purposes.  
  - Configuration: No parameters; used to simulate input.  
  - Connections: Outputs to `Test Input (Change Me)1`.  
  - Edge Cases: No input validation; testing only.

- **for real-time replies**  
  - Type: Webhook  
  - Role: Receives real-time incoming messages at a specific HTTP endpoint.  
  - Configuration: Path set to a unique identifier.  
  - Connections: Feeds into `Test Input (Change Me)1`.  
  - Edge Cases: Webhook downtime or invalid payloads.

- **(5–10 min) for missed or queued**  
  - Type: Schedule Trigger  
  - Role: Periodically triggers the workflow every 5–10 minutes to process queued/missed messages.  
  - Configuration: Interval set to default (every minute placeholder, can be adjusted).  
  - Connections: No direct output connections defined in JSON (likely triggering other workflows or future extension).  
  - Edge Cases: Overlapping executions if processing takes longer than interval.

- **Test Input (Change Me)1**  
  - Type: Set  
  - Role: Prepares the input JSON with a sample customer message text.  
  - Configuration: Sets raw JSON with a sample complaint text.  
  - Connections: Outputs to `Set Config1`.  
  - Edge Cases: Hardcoded text used for testing; requires replacement for live data.

---

#### 2.2 Configuration Setup

**Overview:**  
Defines static parameters controlling the AI response behavior and risk detection.

**Nodes Involved:**  
- `Set Config1` (Set Node)

**Node Details:**

- **Set Config1**  
  - Type: Set  
  - Role: Sets configuration parameters such as maximum reply length, formality, emoji allowance, risk words, and reply style options.  
  - Configuration:  
    - `MAX_LEN`: 600 characters max reply length  
    - `ADD_FOLLOWUP_QUESTION`: true (appends a gentle question to replies)  
    - `FORMALITY`: "auto" (tone formality auto-selected)  
    - `EMOJI_ALLOWED`: false (emojis removed from replies)  
    - `SIGNOFF`: "" (no signoff appended by default)  
    - `BLOCK_LINKS`: true (links removed from replies)  
    - `RISK_WORDS`: Array of keywords indicating high-risk topics (e.g. refund, lawsuit, harassment, suicide)  
  - Connections: Inputs from `Test Input (Change Me)1`; outputs to `Ensure Session1`.  
  - Edge Cases: Misconfiguration risks (e.g., empty or incorrect risk words) may affect routing.

---

#### 2.3 AI Processing

**Overview:**  
Uses an AI agent powered by Anthropic's Claude to analyze input sentiment, tone, and generate an empathetic reply draft.

**Nodes Involved:**  
- `Ensure Session1` (Code)  
- `AI Agent (Empathy)1` (Langchain Agent)  
- `Anthropic Chat Model1` (Anthropic LM)  
- `Memory (Recent 4)` (Memory Buffer)  
- `Structured Output Parser` (Output Parser)

**Node Details:**

- **Ensure Session1**  
  - Type: Code  
  - Role: Normalizes input data by ensuring a session ID and extracting input text.  
  - Configuration: Extracts `sessionId` (defaults to 'test-session-1') and message text.  
  - Inputs: From `Set Config1`.  
  - Outputs: To `AI Agent (Empathy)1`.  
  - Edge Cases: Missing text or session ID fallback to test defaults.

- **AI Agent (Empathy)1**  
  - Type: Langchain Agent  
  - Role: Implements AI prompt logic for analyzing input and generating replies using the Claude model.  
  - Configuration:  
    - System message defines role ("Empathy Reply Assistant") and constraints (English only, no URLs/hashtags).  
    - Prompt includes: sentiment detection, tone selection, reply drafting (max length), and optional follow-up question.  
    - Output: Structured JSON with keys `sentiment`, `tone`, `reply`, `confidence`, `needs_handover`.  
  - Inputs: Receives normalized input text; uses Anthropic LM and Memory nodes as AI model and context.  
  - Outputs: Raw AI-generated JSON output to `Post-Process & Sanitize1`.  
  - Edge Cases: AI model latency, incorrect or malformed output, rate limiting.

- **Anthropic Chat Model1**  
  - Type: Langchain Anthropic LM  
  - Role: Provides the Claude AI model interface (selected model: "claude-3-5-sonnet-latest").  
  - Configuration: Model version locked as latest Sonnet.  
  - Inputs: Linked as AI language model for the AI Agent node.  
  - Outputs: AI text generation results.  
  - Edge Cases: API failures, credential issues.

- **Memory (Recent 4)**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains a short conversation context window (last 4 messages) for the AI agent.  
  - Configuration: `contextWindowLength` set to 4.  
  - Inputs: AI Agent memory interface.  
  - Outputs: Context for AI Agent node.  
  - Edge Cases: Memory overflow or stale context.

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI text output into structured JSON format with schema validation.  
  - Configuration: JSON schema example defines expected keys and types.  
  - Inputs: AI Agent output parser interface.  
  - Outputs: Clean JSON for downstream processing.  
  - Edge Cases: Parsing failures if AI output deviates from schema.

---

#### 2.4 Output Parsing and Sanitization

**Overview:**  
Sanitizes AI-generated reply by removing links, emojis, personal info, trimming length, and optionally appending signoff.

**Nodes Involved:**  
- `Post-Process & Sanitize1` (Code)

**Node Details:**

- **Post-Process & Sanitize1**  
  - Type: Code  
  - Role: Applies content sanitization rules based on config parameters.  
  - Configuration details (derived from `Set Config1`):  
    - Removes URLs and hashtags if `BLOCK_LINKS` is true.  
    - Removes emojis if `EMOJI_ALLOWED` is false.  
    - Redacts email addresses and phone numbers with placeholders.  
    - Truncates reply to `MAX_LEN`.  
    - Appends signoff if configured and not already present.  
  - Inputs: JSON output from AI Agent.  
  - Outputs: Sanitized JSON for risk assessment.  
  - Edge Cases: Regex failures, trimming cutting off words mid-way.

---

#### 2.5 Risk Assessment & Escalation Decision

**Overview:**  
Evaluates the reply and original message for risk keywords, sentiment negativity, and confidence threshold to decide if human handover is necessary.

**Nodes Involved:**  
- `Risk & Handover Rules1` (Code)  
- `If needs_handover1` (If Node)

**Node Details:**

- **Risk & Handover Rules1**  
  - Type: Code  
  - Role: Implements logic combining keyword matching, sentiment, and confidence to flag risky or uncertain replies.  
  - Configuration:  
    - Reads `RISK_WORDS` array from config.  
    - Checks if original message or AI reply contains any risk keywords (regex, case-insensitive).  
    - Flags if confidence < 0.45.  
    - Flags if sentiment is "negative".  
  - Outputs: Adds boolean `needs_handover` and debug info.  
  - Edge Cases: Regex misfires, missing config data.

- **If needs_handover1**  
  - Type: If  
  - Role: Routes the message based on `needs_handover` boolean.  
  - Configuration: Condition checks if `needs_handover` is true.  
  - Outputs:  
    - True branch: to `Draft (Needs Review)`  
    - False branch: to `Draft (Auto-OK)`  
  - Edge Cases: Missing or malformed `needs_handover` field.

---

#### 2.6 Routing & Final Output

**Overview:**  
Marks message status for review or approval; optionally updates Slack or Google Sheets for logging.

**Nodes Involved:**  
- `Draft (Needs Review)` (Set)  
- `Draft (Auto-OK)` (Set)  
- `Update a message` (Slack integration, configured but no channel selected)  
- `Save to Google Sheets1` (Google Sheets logging)

**Node Details:**

- **Draft (Needs Review)**  
  - Type: Set  
  - Role: Marks message with status `REVIEW` and note indicating escalation due to risk or confidence issues.  
  - Configuration: Sets fields `status` = "REVIEW", `note` = "Escalate to human: risk/low-confidence/negative tone detected."  
  - Inputs: From `If needs_handover1` true branch.  
  - Outputs: Final output or further processing (not defined).  
  - Edge Cases: None.

- **Draft (Auto-OK)**  
  - Type: Set  
  - Role: Marks message with status `OK` indicating safe to send automatically.  
  - Configuration: Sets field `status` = "OK".  
  - Inputs: From `If needs_handover1` false branch.  
  - Outputs: Final output or further processing (not defined).  
  - Edge Cases: None.

- **Update a message**  
  - Type: Slack  
  - Role: Intended to update Slack messages with latest draft (integration setup present but no channel selected).  
  - Configuration: Slack credentials via webhook. Operation set to "update".  
  - Inputs: Not connected in workflow.  
  - Edge Cases: Missing channel ID means no updates occur.

- **Save to Google Sheets1**  
  - Type: Google Sheets  
  - Role: Logs workflow metadata to a Google Sheet for tracking or analytics.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Sheet and document IDs read from config parameters (missing in JSON but expected).  
  - Credentials: Uses Google Sheets OAuth2 credentials.  
  - Inputs: Not connected in workflow.  
  - Edge Cases: Missing config parameters, credential errors.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                            | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                             |
|-------------------------|----------------------------------|--------------------------------------------|----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------|
| for testing             | Manual Trigger                   | Manual input trigger for testing           |                            | Test Input (Change Me)1    |                                                                                                       |
| for real-time replies   | Webhook                         | Receives real-time messages                 |                            | Test Input (Change Me)1    |                                                                                                       |
| (5–10 min) for missed or queued | Schedule Trigger                | Scheduled trigger for missed/queued messages |                            |                           |                                                                                                       |
| Test Input (Change Me)1 | Set                             | Sets sample input message                    | for testing, for real-time replies | Set Config1               |                                                                                                       |
| Set Config1             | Set                             | Defines configuration parameters             | Test Input (Change Me)1    | Ensure Session1            | Setup instructions: "Connect Anthropic, edit Set Config1, choose trigger"                             |
| Ensure Session1         | Code                            | Normalizes input text and session ID        | Set Config1                | AI Agent (Empathy)1        |                                                                                                       |
| AI Agent (Empathy)1     | Langchain Agent                 | AI sentiment analysis and reply generation  | Ensure Session1, Anthropic Chat Model1, Memory (Recent 4), Structured Output Parser | Post-Process & Sanitize1  | Empathy Reply Assistant description: generates empathetic replies, sanitizes, auto-escalates          |
| Anthropic Chat Model1   | Langchain LM                   | Provides Claude AI model interface           |                            | AI Agent (Empathy)1 (ai_languageModel) | Setup instructions: "Connect Anthropic node"                                                          |
| Memory (Recent 4)       | Langchain Memory Buffer         | Maintains recent conversation context       |                            | AI Agent (Empathy)1 (ai_memory) |                                                                                                       |
| Structured Output Parser| Langchain Output Parser         | Parses AI output into structured JSON        |                            | AI Agent (Empathy)1 (ai_outputParser) |                                                                                                       |
| Post-Process & Sanitize1| Code                            | Cleans reply: remove links, emojis, PII     | AI Agent (Empathy)1        | Risk & Handover Rules1     |                                                                                                       |
| Risk & Handover Rules1  | Code                            | Flags risky or low-confidence replies       | Post-Process & Sanitize1   | If needs_handover1         | Routes & Thresholds note: escalate on negative, risk words, low confidence                            |
| If needs_handover1      | If                              | Routes based on need for human handover     | Risk & Handover Rules1     | Draft (Needs Review), Draft (Auto-OK) |                                                                                                       |
| Draft (Needs Review)    | Set                             | Marks message as needing human review       | If needs_handover1 (true)  |                           | Routes & Thresholds note: REVIEW status / escalation                                                  |
| Draft (Auto-OK)         | Set                             | Marks message as OK to send automatically   | If needs_handover1 (false) |                           | Routes & Thresholds note: OK status                                                                  |
| Update a message        | Slack                           | (Configured but unused) Updates Slack message |                           |                           |                                                                                                       |
| Save to Google Sheets1  | Google Sheets                   | (Configured but unused) Logs data to Sheets |                           |                           |                                                                                                       |
| Sticky Note             | Sticky Note                    | Documentation overview                       |                            |                           | Empathy Reply Assistant overview and flow description                                                |
| Sticky Note1            | Sticky Note                    | Documentation on routing & thresholds        |                            |                           | Escalation and auto-OK criteria                                                                        |
| Sticky Note2            | Sticky Note                    | Setup instructions                           |                            |                           | Setup (3-5 min) steps for Anthropic, config, triggers                                                |
| Sticky Note3            | Sticky Note                    | Notes and safety instructions                |                            |                           | Safety notes, no hardcoded keys, draft status, author & license info                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Nodes**  
   - Add a **Manual Trigger** node named `for testing`. No parameters.  
   - Add a **Webhook** node named `for real-time replies`. Set path to a unique string (e.g., "7a62325a-0000-4724-8fe4-8829e3dea2fb").  
   - Add a **Schedule Trigger** node named `(5–10 min) for missed or queued`. Configure interval as desired (e.g., every 5 minutes).  

2. **Add Input Preparation Node**  
   - Add a **Set** node named `Test Input (Change Me)1`.  
   - Configure mode to "Raw JSON" with:  
     ```json
     {
       "text": "I'm really frustrated with your service. This is the third time I've had issues and nobody is helping me!"
     }
     ```  
   - Connect outputs of `for testing` and `for real-time replies` to this node. (Schedule trigger is not connected in this JSON but can be configured to feed here or another input node.)

3. **Add Configuration Node**  
   - Add a **Set** node named `Set Config1`.  
   - Configure raw JSON with:  
     ```json
     {
       "MAX_LEN": 600,
       "ADD_FOLLOWUP_QUESTION": true,
       "FORMALITY": "auto",
       "EMOJI_ALLOWED": false,
       "SIGNOFF": "",
       "BLOCK_LINKS": true,
       "RISK_WORDS": [
         "refund","chargeback","lawsuit","harassment","self-harm","suicide","abuse",
         "threat","racist","illegal","hate","scam"
       ]
     }
     ```  
   - Connect output of `Test Input (Change Me)1` to `Set Config1`.

4. **Normalize Input Text**  
   - Add a **Code** node named `Ensure Session1`.  
   - Use this JavaScript code:  
     ```javascript
     const j = $input.first().json || {};
     const sid = j.sessionId || 'test-session-1';
     const text = j.text || j.message || j.input || JSON.stringify(j);
     return [{ json: { ...j, sessionId: String(sid), text } }];
     ```  
   - Connect `Set Config1` output to `Ensure Session1`.

5. **Add AI Processing Nodes**  
   - **Anthropic Chat Model node:**  
     - Add a Langchain Anthropic LM node named `Anthropic Chat Model1`.  
     - Select model: `claude-3-5-sonnet-latest` (or current latest).  
     - Set credentials with Anthropic API key stored securely in n8n Credentials.  
   - **Memory Buffer node:**  
     - Add Langchain Memory Buffer Window node named `Memory (Recent 4)`.  
     - Set context window length to 4.  
   - **Output Parser node:**  
     - Add Langchain Output Parser Structured node named `Structured Output Parser`.  
     - Define JSON schema example as:  
       ```json
       {
         "sentiment": "positive|neutral|negative|mixed",
         "tone": "warm|calm|upbeat|apologetic|confident|concise",
         "reply": "string",
         "confidence": 0.0,
         "needs_handover": false
       }
       ```  
   - **AI Agent node:**  
     - Add Langchain Agent node named `AI Agent (Empathy)1`.  
     - Set prompt with the text below (use n8n expression syntax for config parameters):  
       ```
       You are an Empathy Reply Assistant for English-language messages.
       Analyze the incoming text:
       <Text>{{ $json.text || $json.message || $json.input || JSON.stringify($json) }}</Text>

       Goals:
       1) Detect `sentiment` (positive|neutral|negative|mixed)
       2) Choose a natural human `tone` that fits the sender (warm, calm, upbeat, apologetic, confident, concise)
       3) Draft a short `reply` (<= {{ $('Set Config1').item.json.MAX_LEN }} chars) that:
          - acknowledges the emotion first in plain English
          - provides a practical next step or clarification
          - {{ $('Set Config1').item.json.ADD_FOLLOWUP_QUESTION ? 'ends with a gentle question' : 'may end without a question' }}
          - uses **no links**, **no hashtags**
          - matches formality: {{ $('Set Config1').item.json.FORMALITY }} (auto|casual|polite)

       Return **JSON only** with keys: sentiment, tone, reply, confidence (0.0–1.0), needs_handover (boolean).
       ```  
     - Set system message:  
       ```
       Role: Empathetic, practical, concise. Avoid corporate cliches. Keep it in English. No URLs or hashtags.
       ```  
     - Attach Anthropic Chat Model node as AI language model.  
     - Attach Memory (Recent 4) as AI memory.  
     - Attach Structured Output Parser as AI output parser.  
     - Connect `Ensure Session1` output to AI Agent input.

6. **Add Sanitization Node**  
   - Add a **Code** node named `Post-Process & Sanitize1`.  
   - Use the provided JavaScript code that:  
     - Reads config from `Set Config1`  
     - Removes links, hashtags, emojis (if disabled)  
     - Redacts emails and phone numbers  
     - Truncates reply to max length  
     - Optionally appends signoff  
   - Connect output of `AI Agent (Empathy)1` to this node.

7. **Add Risk Assessment and Routing Nodes**  
   - Add a **Code** node named `Risk & Handover Rules1`.  
   - Use the JavaScript code that:  
     - Reads `RISK_WORDS` from config  
     - Checks for risk keywords and negative sentiment  
     - Flags low confidence (<0.45)  
     - Sets `needs_handover` boolean accordingly  
   - Connect `Post-Process & Sanitize1` output to this node.  
   - Add an **If** node named `If needs_handover1`.  
   - Condition: `{{$json.needs_handover}} === true`.  
   - Connect `Risk & Handover Rules1` output to this node.

8. **Add Final Status Marking Nodes**  
   - Add a **Set** node named `Draft (Needs Review)`.  
   - Set fields:  
     - `status`: `"REVIEW"`  
     - `note`: `"Escalate to human: risk/low-confidence/negative tone detected."`  
   - Connect `If needs_handover1` True branch here.  
   - Add a **Set** node named `Draft (Auto-OK)`.  
   - Set field:  
     - `status`: `"OK"`  
   - Connect `If needs_handover1` False branch here.

9. **(Optional) Add Slack and Google Sheets Logging**  
   - Add a Slack node named `Update a message` configured with Slack credentials and operation “update” (channel ID required).  
   - Add a Google Sheets node named `Save to Google Sheets1` configured to append or update rows with relevant metadata. Use OAuth2 credentials.  
   - These nodes are optional and not connected by default.

10. **Add Sticky Notes for Documentation**  
    - Add sticky notes with content describing workflow overview, routing thresholds, setup instructions, and safety notes as per original content.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Empathy Reply Assistant generates empathetic reply drafts, sanitizes content, and auto-escalates risky cases.           | Workflow purpose and flow overview sticky note                                                  |
| Routes & Thresholds: Escalate on negative sentiment, risk words, low confidence; Auto-OK otherwise; outputs status.    | Routing and escalation criteria sticky note                                                    |
| Setup instructions: Connect Anthropic node, configure Set Config1 parameters, choose trigger method (manual/webhook). | Setup sticky note                                                                              |
| Safety notes: No hardcoded API keys; treat outputs as drafts; remove test data before sharing; tune thresholds.        | Safety and author/license sticky note                                                          |
| Author: Yusuke (@yskautomation)                                                                                       | Credit                                                                                         |
| License: MIT                                                                                                          | License info                                                                                   |
| Anthropic Claude model used: claude-3-5-sonnet-latest                                                                 | AI model details                                                                              |
| Keep sensitive data and API keys securely in n8n Credentials, not in nodes.                                           | Security best practices                                                                         |

---

**Disclaimer:** The provided text and workflow are entirely generated and managed within an automated n8n environment with strict compliance to content policies. No illegal, offensive, or protected data is processed.