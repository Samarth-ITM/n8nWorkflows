🌲 AI Agent for Sustainability Report Audit with Gmail and GPT-40

https://n8nworkflows.xyz/workflows/---ai-agent-for-sustainability-report-audit-with-gmail-and-gpt-40-3420


# 🌲 AI Agent for Sustainability Report Audit with Gmail and GPT-40

### 1. Workflow Overview

This workflow automates the auditing of Corporate Sustainability Reporting Directive (CSRD) XHTML reports received via Gmail. It is designed for sustainability teams, ESG consultants, and developers who need to validate the structure and content of CSRD reports to ensure compliance with regulatory standards.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Filtering:** Triggered by incoming Gmail messages, it filters emails based on the subject line to identify relevant CSRD reports.
- **1.2 XHTML Extraction and Parsing:** Downloads the email attachment, extracts the XHTML content, and performs structural validation and content checks on the report.
- **1.3 AI-Powered Audit Summary Generation:** Uses an AI agent (LangChain with OpenAI GPT-4o-mini) to generate a professional email summarizing the audit results.
- **1.4 Email Reply with Audit Report:** Sends the AI-generated audit summary as a reply to the original email sender.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block listens for new emails in Gmail and filters them to process only those related to CSRD reporting by checking if the subject contains "CSRD Reporting".

**Nodes Involved:**  
- Email Trigger (Gmail Trigger)  
- If (Conditional Filter)

**Node Details:**

- **Email Trigger (Gmail Trigger)**  
  - Type: Trigger node for Gmail  
  - Role: Listens for new incoming emails in the connected Gmail account  
  - Configuration: Polls every minute; requires Gmail API credentials  
  - Input: None (trigger)  
  - Output: Email metadata and content JSON  
  - Edge Cases: Gmail API quota limits, authentication errors, network failures  
  - Sticky Note: Explains setup and purpose with link to Gmail Trigger documentation

- **If (Conditional Filter)**  
  - Type: Conditional node  
  - Role: Checks if the email subject contains the string "CSRD Reporting"  
  - Configuration: Uses a string "contains" operation on the subject field (`{{$json.subject}}`)  
  - Input: Output from Email Trigger  
  - Output: Passes emails matching the condition to the next block; ignores others  
  - Edge Cases: Case sensitivity is enabled; emails without subject or malformed JSON could cause failures

---

#### 2.2 XHTML Extraction and Parsing

**Overview:**  
This block downloads the email attachment (expected to be the CSRD XHTML report), extracts the XHTML content as text, and runs a custom JavaScript audit to validate the presence and correctness of key tags and disclosures.

**Nodes Involved:**  
- Download Attachment (Gmail)  
- HTML from binary (Extract from File)  
- Extract the HTML (Code)  
- Check the format (Code)

**Node Details:**

- **Download Attachment (Gmail)**  
  - Type: Gmail node (operation: get)  
  - Role: Downloads attachments from the email identified by message ID  
  - Configuration: Downloads all attachments; requires Gmail API credentials  
  - Input: Email metadata from If node  
  - Output: Binary data of attachments  
  - Edge Cases: Missing attachments, large files, API errors

- **HTML from binary (Extract from File)**  
  - Type: Extract from File node  
  - Role: Converts the binary attachment (expected XHTML file) into text content  
  - Configuration: Operation set to "text", binary property name set to "attachment_0"  
  - Input: Binary attachment from Download Attachment  
  - Output: JSON with extracted XHTML content as text  
  - Edge Cases: Binary data corrupt or not a valid text file

- **Extract the HTML (Code)**  
  - Type: Code node (JavaScript)  
  - Role: Wraps the extracted text content into a JSON property named `xhtml_content` for downstream processing  
  - Configuration: Simple return of the input content under a new JSON key  
  - Input: Text content from Extract from File  
  - Output: JSON with `xhtml_content` property  
  - Edge Cases: Empty or missing content

- **Check the format (Code)**  
  - Type: Code node (JavaScript)  
  - Role: Performs detailed audit of the XHTML content by parsing for required tags and checking for compliance indicators  
  - Configuration:  
    - Uses regex to extract `<ix:nonNumeric>` tags and other key tags such as `<ix:header>`, `esrs:SustainabilityGovernance`, and `esrs:StrategySustainability`.  
    - Checks for presence of key KPIs (GHG emissions scopes 1, 2, 3).  
    - Detects empty disclosures and duplicate values.  
    - Returns a JSON object summarizing audit results with counts and pass/missing flags.  
  - Input: JSON with `xhtml_content` from previous node  
  - Output: JSON audit results with detailed metrics  
  - Edge Cases: Malformed XHTML, missing tags, regex failures, large content performance issues  
  - Sticky Note: Explains the audit logic and setup

---

#### 2.3 AI-Powered Audit Summary Generation

**Overview:**  
This block uses an AI agent to convert the raw JSON audit results into a professional, readable email summary addressed to the sustainability team.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- OpenAI Chat Model (GPT-4o-mini)  
- Structured Output Parser

**Node Details:**

- **AI Agent (LangChain Agent)**  
  - Type: AI agent node integrating LangChain with OpenAI  
  - Role: Orchestrates prompt construction and response parsing for generating the audit summary email  
  - Configuration:  
    - Input text includes JSON stringified audit results.  
    - System message defines the assistant’s role as an ESG compliance assistant writing clear, actionable email summaries.  
    - Output expected in JSON format with `subject` and `body` fields.  
  - Input: Audit results JSON from Check the format node  
  - Output: Parsed JSON with email subject and body  
  - Edge Cases: API rate limits, malformed JSON input, AI model errors, prompt misinterpretation  
  - Requires OpenAI API credentials  
  - Sticky Note: Setup instructions and best practices for AI Agent and Chat Model

- **OpenAI Chat Model (GPT-4o-mini)**  
  - Type: Language model node for OpenAI GPT-4o-mini  
  - Role: Provides the natural language generation capability for the AI Agent  
  - Configuration: Model set to "gpt-4o-mini"  
  - Input: Prompt from AI Agent  
  - Output: Raw AI-generated text  
  - Edge Cases: API quota, network issues

- **Structured Output Parser**  
  - Type: Output parser node for LangChain  
  - Role: Parses AI output to enforce JSON structure with `subject` and `body` keys  
  - Configuration: JSON schema example provided to validate output format  
  - Input: Raw AI output from OpenAI Chat Model  
  - Output: Structured JSON for downstream use  
  - Edge Cases: Parsing errors if AI output deviates from expected format

---

#### 2.4 Email Reply with Audit Report

**Overview:**  
This final block sends the AI-generated audit summary as a reply to the original email sender, closing the loop on the audit process.

**Nodes Involved:**  
- Reply (Gmail)

**Node Details:**

- **Reply (Gmail)**  
  - Type: Gmail node (operation: reply)  
  - Role: Sends a reply email with the audit summary to the original sender  
  - Configuration:  
    - Uses the message ID of the original email to reply in-thread.  
    - Email body is plain text from AI Agent output.  
    - Requires Gmail API credentials  
  - Input: Structured JSON from AI Agent node containing `body` and `subject`  
  - Output: Confirmation of email sent  
  - Edge Cases: Gmail API errors, invalid message ID, authentication failures  
  - Sticky Note: Setup instructions for Gmail node

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                      |
|---------------------|----------------------------------|---------------------------------------|------------------------|-----------------------|------------------------------------------------------------------------------------------------|
| Email Trigger       | Gmail Trigger                    | Triggers workflow on new emails       | None                   | If                    | Explains Gmail Trigger setup and filtering by subject "CSRD Reporting"                          |
| If                  | Conditional                     | Filters emails by subject content     | Email Trigger          | Download Attachment    |                                                                                                |
| Download Attachment  | Gmail                           | Downloads email attachments            | If                     | HTML from binary      |                                                                                                |
| HTML from binary     | Extract from File               | Converts binary attachment to text    | Download Attachment     | Extract the HTML       |                                                                                                |
| Extract the HTML     | Code                           | Wraps XHTML content into JSON          | HTML from binary        | Check the format       |                                                                                                |
| Check the format     | Code                           | Audits XHTML content for compliance   | Extract the HTML        | AI Agent              | Explains audit logic and parsing rules                                                        |
| AI Agent            | LangChain Agent                 | Generates audit summary email          | Check the format        | Reply                 | Setup instructions for AI Agent and Chat Model with OpenAI GPT-4o-mini                         |
| OpenAI Chat Model    | LangChain LM Chat OpenAI        | Provides AI natural language generation| AI Agent (ai_languageModel) | AI Agent (ai_outputParser) |                                                                                                |
| Structured Output Parser | LangChain Output Parser       | Parses AI output into structured JSON  | OpenAI Chat Model       | AI Agent              |                                                                                                |
| Reply                | Gmail                          | Sends reply email with audit summary  | AI Agent                | None                  | Setup instructions for Gmail reply node                                                       |
| Sticky Note1         | Sticky Note                    | Documentation for block 1              | None                   | None                  | Covers Email Trigger and If node setup                                                        |
| Sticky Note          | Sticky Note                    | Documentation for block 2              | None                   | None                  | Covers Download Attachment to Check the format nodes                                          |
| Sticky Note2         | Sticky Note                    | Documentation for block 3              | None                   | None                  | Covers AI Agent, OpenAI Chat Model, Structured Output Parser nodes                             |
| Sticky Note3         | Sticky Note                    | Additional resources and tutorial link | None                   | None                  | Provides external tutorial video link                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set credentials for Gmail API access  
   - Configure polling interval (e.g., every minute)  
   - No filters needed here; filtering done downstream  
   - Position: Start of workflow

2. **Add If Node for Subject Filtering**  
   - Type: If  
   - Condition: Check if `{{$json.subject}}` contains string "CSRD Reporting" (case sensitive)  
   - Connect Gmail Trigger output to If node input  
   - True branch proceeds; False branch ends workflow

3. **Add Gmail Node to Download Attachments**  
   - Type: Gmail  
   - Operation: Get message  
   - Parameters:  
     - Message ID: `{{$json.id}}` from If node output  
     - Download Attachments: Enabled  
   - Connect If node "true" output to this node

4. **Add Extract from File Node to Convert Binary to Text**  
   - Type: Extract from File  
   - Operation: Text  
   - Binary Property Name: `attachment_0` (first attachment)  
   - Connect Download Attachment node output to this node

5. **Add Code Node to Wrap XHTML Content**  
   - Type: Code  
   - JavaScript:  
     ```js
     return [{
       json: {
         xhtml_content: $input.first().json.data
       }
     }];
     ```  
   - Connect Extract from File node output to this node

6. **Add Code Node to Audit XHTML Content**  
   - Type: Code  
   - JavaScript: Use the provided audit script that:  
     - Extracts `<ix:nonNumeric>` tags  
     - Checks for `<ix:header>`, `esrs:SustainabilityGovernance`, `esrs:StrategySustainability` tags  
     - Detects KPI tags for GHG emissions scopes 1, 2, 3  
     - Counts empty disclosures and duplicates  
     - Returns JSON audit summary  
   - Connect previous Code node output to this node

7. **Add LangChain AI Agent Node**  
   - Type: LangChain Agent  
   - Configure with OpenAI API credentials  
   - Set input text to:  
     ```
     Generate an email to the sustainability team summarizing this CSRD XHTML report audit:

     {{JSON.stringify($json.audit_results, null, 2)}}

     Return the output in the following JSON format:

     {
       "subject": "...",
       "body": "..."
     }
     ```  
   - Set system message to define the assistant role as ESG compliance assistant writing professional email summaries  
   - Connect audit Code node output to AI Agent input

8. **Add OpenAI Chat Model Node**  
   - Type: LangChain LM Chat OpenAI  
   - Model: gpt-4o-mini  
   - Connect AI Agent node’s language model input to this node

9. **Add Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - JSON schema example:  
     ```json
     {
       "subject": "CSRD XHTML Report Audit – Key Findings and Next Steps",
       "body": "Content of the email"
     }
     ```  
   - Connect OpenAI Chat Model output to this node  
   - Connect this node output back to AI Agent output parser input

10. **Add Gmail Node to Reply to Email**  
    - Type: Gmail  
    - Operation: Reply  
    - Message ID: `{{$json.id}}` from original email  
    - Message: `{{$json.output.body}}` from AI Agent output  
    - Email Type: Text  
    - Connect AI Agent output to this node

11. **Add Sticky Notes** (Optional but recommended)  
    - Add notes describing each block’s purpose and setup instructions, including links to documentation and tutorials.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow created by Samir, founder of LogiGreen Consulting, expert in sustainability automation | https://www.logi-green.com/                                                                      |
| CSRD XHTML report format explanation with XBRL tagging requirements                          | https://www.samirsaci.com/content/images/2025/04/temp.png                                        |
| Tutorial video explaining workflow usage and setup                                          | https://youtu.be/npeJZv5U7og                                                                     |
| GitHub repository with sample XHTML files for testing                                       | https://github.com/samirsaci/n8n-templates                                                      |
| Gmail Trigger Node documentation                                                            | https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger                |
| AI Agent Node documentation                                                                 | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent       |

---

This document provides a detailed, structured reference to understand, reproduce, and extend the "AI Agent for Sustainability Report Audit with Gmail and GPT-40" workflow in n8n. It covers all nodes, their configurations, interconnections, and potential failure points, enabling both human users and AI agents to work effectively with this automation.