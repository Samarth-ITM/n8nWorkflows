Validate Newsletter Quality with GPT-5 Quality Gate before Sending

https://n8nworkflows.xyz/workflows/validate-newsletter-quality-with-gpt-5-quality-gate-before-sending-10734


# Validate Newsletter Quality with GPT-5 Quality Gate before Sending

### 1. Workflow Overview

This workflow performs **automated quality assurance (QA) on a newsletter email before it is sent to subscribers**, using a GPT-5-powered "LLM Judge" as a final gatekeeper. Its goal is to prevent broken, incomplete, or poorly formatted newsletters from reaching customers by simulating how the newsletter renders in Gmail and applying AI-driven validation rules.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Input Reception**  
  Receives the generated newsletter HTML content from a parent workflow.

- **1.2 Send Newsletter to LLM Judge**  
  Sends the newsletter HTML to a dedicated Gmail inbox ("LLM Judge") for rendering.

- **1.3 Retrieve Rendered Email**  
  Waits briefly, then fetches the actual rendered email back from Gmail to capture any Gmail-specific rendering issues.

- **1.4 Prepare and Submit to AI Judge**  
  Simplifies the fetched email content and sends it to the GPT-5-based AI Judge for quality validation, using a detailed system prompt.

- **1.5 Parse AI Response and Decision Gate**  
  Parses the structured response from the AI Judge, then routes the workflow based on the verdict:
    - **PASS**: Approves sending the newsletter to customers.
    - **BLOCK**: Sends an alert email to the admin requesting manual review.

- **1.6 Error and Manual Review Handling**  
  Handles failures or BLOCK decisions by notifying a human administrator for intervention.

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger and Input Reception

- **Overview:**  
  Entry point triggered by a parent newsletter workflow passing the newsletter HTML for review.

- **Nodes Involved:**  
  - Executed from Newsletter Creation workflow  
  - Sticky Note (explains trigger context)

- **Node Details:**  
  - **Executed from Newsletter Creation workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Receives newsletter HTML content as passthrough input from main workflow.  
    - Configuration: Input source set to passthrough, no credentials needed.  
    - Input: Triggered externally, expects JSON with `html` field (newsletter content).  
    - Output: Passes newsletter HTML downstream.  
    - Edge Cases: Trigger failure if input missing or malformed HTML.  

  - **Sticky Note**  
    - Purpose: Documentation only, no runtime effect.

---

#### 2.2 Send Newsletter to LLM Judge

- **Overview:**  
  Sends the newsletter HTML content to the LLM Judge Gmail inbox to generate a real rendering of the email.

- **Nodes Involved:**  
  - Send newsletter to LLM Judge  
  - Sticky Note2 (explains sending action)

- **Node Details:**  
  - **Send newsletter to LLM Judge**  
    - Type: Gmail node (send email)  
    - Role: Sends a test email containing the newsletter HTML to the Judge's Gmail address.  
    - Configuration:  
      - `sendTo` fixed to "emir.belkahia@gmail.com" (Judge's inbox).  
      - Subject prefixed with "FOR REVIEW BY JUDGE".  
      - Message body is the newsletter HTML passed from trigger node.  
      - Attribution disabled (no extra footer).  
    - Credentials: Gmail OAuth2 account connected.  
    - Input: Receives newsletter HTML JSON from trigger.  
    - Output: Email metadata including message ID for retrieval.  
    - Edge Cases: Gmail auth failure, network issues, invalid HTML content.  

  - **Sticky Note2**  
    - Describes the purpose: sending newsletter to Judge's mailbox for rendering.

---

#### 2.3 Retrieve Rendered Email

- **Overview:**  
  Waits for the test email to arrive and fetches it back from Gmail to analyze how Gmail renders it.

- **Nodes Involved:**  
  - Give the email some time to arrive  
  - Get the newsletter  
  - Sticky Note3 (explains retrieval purpose)

- **Node Details:**  
  - **Give the email some time to arrive**  
    - Type: Wait node  
    - Role: Introduces a delay to allow Gmail delivery before fetching.  
    - Configuration: Default wait time (seconds) - not explicitly set, so defaults apply.  
    - Edge Cases: Insufficient wait time may cause fetch failure or incomplete email.  

  - **Get the newsletter**  
    - Type: Gmail node (get email)  
    - Role: Retrieves the email using the message ID from "Send newsletter to LLM Judge".  
    - Configuration:  
      - Operation: "get" by message ID.  
      - Uses message ID from previous node dynamically via expression.  
    - Credentials: Same Gmail OAuth2 as sender node.  
    - Input: Message ID from send node output.  
    - Output: Full email object including rendered HTML.  
    - Edge Cases: Email not found, Gmail API errors, auth issues.  

  - **Sticky Note3**  
    - Explains that this round-trip captures Gmail-specific rendering issues like CSS stripping or image blocking.

---

#### 2.4 Prepare and Submit to AI Judge

- **Overview:**  
  Simplifies the retrieved email content to isolate the HTML and submits it to the GPT-5-powered AI Judge for quality validation.

- **Nodes Involved:**  
  - Simplify output before giving to Agent  
  - Judge 👩‍⚖️ (LangChain agent node)  
  - Expected Format (AI Output Parser)  
  - GPT5 as the input to review is heavy  
  - Sticky Note4 (explains AI Judge role)  
  - Sticky Note13 (explains output format)

- **Node Details:**  
  - **Simplify output before giving to Agent**  
    - Type: Set node  
    - Role: Extracts only the `html` field from the Gmail email object.  
    - Configuration: Sets `html` property to the fetched email’s HTML content via expression.  
    - Input: Full email JSON from "Get the newsletter".  
    - Output: Simplified JSON with only `html`.  
    - Edge Cases: Missing or malformed HTML.  

  - **Judge 👩‍⚖️**  
    - Type: LangChain Agent node (OpenAI GPT-based)  
    - Role: Runs the core quality assurance check on newsletter HTML.  
    - Configuration:  
      - Uses a detailed system prompt that describes comprehensive QA checks and decision rules for the newsletter.  
      - Input text includes the rendered HTML email.  
      - Output is a structured JSON verdict with status, confidence, issues found, and recommendations.  
      - OnError set to continue with error output to handle AI failures gracefully.  
    - Credentials: Connected OpenAI API key configured for GPT-5 model.  
    - Input: HTML from "Simplify output before giving to Agent".  
    - Output: AI-assessed JSON report.  
    - Edge Cases: OpenAI API errors, model timeouts, prompt parsing failures.  

  - **Expected Format**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Validates and parses AI Judge’s JSON output according to a strict schema.  
    - Configuration: Schema requires `status` (PASS/BLOCK), `confidence` (int), `issues_found` (array of objects), and `recommendation` (string).  
    - Input: Raw AI output from Judge node.  
    - Output: Parsed and validated structured JSON for downstream use.  
    - Edge Cases: AI output not matching schema, parsing errors.  

  - **GPT5 as the input to review is heavy**  
    - Type: LangChain LM Chat OpenAI node  
    - Role: Invokes GPT-5 model to process the newsletter HTML for Judge node.  
    - Configuration: Model set explicitly to `gpt-5`.  
    - Credentials: OpenAI API key connected.  
    - Input: Newsletter HTML.  
    - Output: AI-generated response fed into Judge node.  
    - Edge Cases: Model availability, rate limits, API errors.  

  - **Sticky Note4**  
    - Summarizes the Judge’s responsibilities: completeness, layout, detecting broken or missing data, final pass/block decision.  

  - **Sticky Note13**  
    - Describes the output format returned by the Judge node.

---

#### 2.5 Parse AI Response and Decision Gate

- **Overview:**  
  Uses the AI Judge's verdict to decide whether to approve the newsletter for sending or block and alert the administrator.

- **Nodes Involved:**  
  - Judge happy ? (IF node)  
  - Send approval to parent workflow  
  - Ask Human to review Newsletter  
  - Inform human that there are errors  
  - Sticky Note5 (decision gate explanation)  
  - Sticky Note1 (workflow error handling)  

- **Node Details:**  
  - **Judge happy ?**  
    - Type: IF node  
    - Role: Checks if the Judge’s `output.status` equals `"PASS"`.  
    - Input: Parsed AI Judge output JSON.  
    - Output:  
      - True branch: newsletter passed QA.  
      - False branch: newsletter blocked or failed QA.  
    - Edge Cases: Missing `output.status`, unexpected status values.  

  - **Send approval to parent workflow**  
    - Type: Set node  
    - Role: Returns the `output.status` back to the parent workflow signaling approval.  
    - Configuration: Sets `output.status` to the Judge’s status value.  
    - Input: True branch from IF node.  
    - Output: Success response to parent workflow.  
    - Edge Cases: Output format mismatch with parent expectations.  

  - **Ask Human to review Newsletter**  
    - Type: Gmail node (send email)  
    - Role: Sends detailed AI Judge’s JSON output to admin for manual review upon BLOCK decision.  
    - Configuration:  
      - Sends to "emir.belkahia@gmail.com" (admin).  
      - Subject indicates newsletter not approved.  
      - Body contains formatted JSON of issues found.  
      - Operation set to send and wait for response (freeText).  
    - Credentials: Gmail OAuth2 connected.  
    - Input: False branch from IF node with AI output JSON.  
    - Output: Admin notification.  
    - Edge Cases: Email send failure, admin mailbox issues.  

  - **Inform human that there are errors**  
    - Type: Gmail node (send email)  
    - Role: Sends alert email to admin if AI Judge node errors (e.g. model failure).  
    - Configuration: Simple notification text with timestamp.  
    - Credentials: Gmail OAuth2 connected.  
    - Input: Error output from Judge node.  
    - Output: Admin alert.  
    - Edge Cases: Email send failure, missing error details.  

  - **Sticky Note5**  
    - Explains decision logic: PASS returns success to parent; BLOCK sends alert email for manual intervention.  

  - **Sticky Note1**  
    - Describes general error handling for agent/model failures requiring admin review.

---

#### 2.6 Documentation and Metadata Notes

- **Nodes Involved:**  
  - Multiple Sticky Notes (Sticky Note9, Sticky Note11, Sticky Note12, Sticky Note13)  

- **Purpose:**  
  Provide extensive documentation within the workflow about newsletter structure, QA process, setup instructions, contact info, and AI output format.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                                | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                                 |
|--------------------------------|----------------------------------|------------------------------------------------|--------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Executed from Newsletter Creation workflow | Execute Workflow Trigger          | Entry point; receives newsletter HTML          | External trigger               | Send newsletter to LLM Judge     | ## Upon Newsletter Generation This sub-workflow gets triggered by a parent workflow which needs a review of the newsletter by the LLM Judge. Previous node sends the HTML content of the newsletter. |
| Send newsletter to LLM Judge   | Gmail (send email)                | Sends newsletter HTML to Judge’s Gmail inbox   | Executed from Newsletter Creation workflow | Give the email some time to arrive | ## Send Prepared Newsletter to LLM Judge's mailbox. Sends the newsletter to the inbox of the LLM Judge so it can grab the rendered version of it. |
| Give the email some time to arrive | Wait                             | Wait for email delivery delay                    | Send newsletter to LLM Judge    | Get the newsletter               |                                                                                                                             |
| Get the newsletter             | Gmail (get email)                 | Retrieves rendered newsletter email from Gmail | Give the email some time to arrive | Simplify output before giving to Agent | ## Retrieve Newsletter from Gmail Fetches the sent newsletter back from Gmail to validate the actual rendered output. This round-trip ensures we catch any Gmail-specific rendering issues (CSS stripping, image blocking, etc.) before sending to customers. Cost: negligible. |
| Simplify output before giving to Agent | Set                              | Extracts HTML content for AI Judge               | Get the newsletter              | GPT5 as the input to review is heavy |                                                                                                                             |
| GPT5 as the input to review is heavy | LangChain LM Chat OpenAI          | Calls GPT-5 model to analyze newsletter HTML    | Simplify output before giving to Agent | Judge 👩‍⚖️                   |                                                                                                                             |
| Judge 👩‍⚖️                     | LangChain Agent                  | Performs AI quality assurance judgment          | GPT5 as the input to review is heavy | Judge happy ?                   | ## Do Newsletter Quality Assurance LLM Judge validates the newsletter for completeness and quality: - Checks all 6 products have images, prices, and descriptions - Verifies layout integrity and date range formatting - Detects broken images or unprocessed template variables Returns: `PASS` to proceed or `BLOCK` with detailed issue report. |
| Expected Format                | LangChain Output Parser Structured | Validates and parses AI output JSON              | Judge 👩‍⚖️                     | Judge 👩‍⚖️ (ai_outputParser)     | ## Judge Output Format Returns: - `status` (PASS/BLOCK) - `confidence` (0-100), `issues_found` (array), - `recommendation` (action to take) |
| Judge happy ?                 | IF                               | Decision gate based on Judge’s PASS/BLOCK status | Judge 👩‍⚖️                     | Send approval to parent workflow (true), Ask Human to review Newsletter (false) | ## Decision Gate - Send or Alert Based on the Judge's decision: ✅ PASS: Returns success to parent workflow → newsletter gets sent to customers 🚨 BLOCK: Sends alert email to admin with issue details → manual review required before sending |
| Send approval to parent workflow | Set                              | Sends PASS status back to parent workflow        | Judge happy ? (true branch)     |                                 |                                                                                                                             |
| Ask Human to review Newsletter | Gmail (send email)                | Sends BLOCK notification and details to admin   | Judge happy ? (false branch)    |                                 |                                                                                                                             |
| Inform human that there are errors | Gmail (send email)                | Sends alert if AI Judge node errors occur        | Judge 👩‍⚖️ (error output)       |                                 | ## Workflow error handling Something went wrong with the Agent (could be the model or the parser). Cause should be reviewed by Admin. |
| Sticky Note                    | Sticky Note                      | Documentation and explanations                    | N/A                            | N/A                             | Various notes on newsletter structure, QA process, setup, contact, and workflow context (see detailed content above).        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Type: Execute Workflow Trigger  
   - Configure input source as passthrough to accept newsletter HTML JSON from parent workflow.

2. **Create Gmail Send Node:**  
   - Name: "Send newsletter to LLM Judge"  
   - Type: Gmail (Send Email)  
   - Configure:  
     - Recipient: set to Judge’s email (e.g., emir.belkahia@gmail.com)  
     - Subject: prefix with "FOR REVIEW BY JUDGE - [Newsletter title]"  
     - Message body: use expression to pass the input newsletter HTML (`{{$json.html}}`)  
     - Disable attribution footer  
   - Connect Gmail OAuth2 credentials.

3. **Add Wait Node:**  
   - Name: "Give the email some time to arrive"  
   - Type: Wait  
   - Configure default wait time (e.g., 10-30 seconds) to allow email delivery.

4. **Create Gmail Get Node:**  
   - Name: "Get the newsletter"  
   - Type: Gmail (Get Email)  
   - Configure operation to "get" email by ID  
   - Use expression to get message ID from previous send node output (`{{$node["Send newsletter to LLM Judge"].json.id}}`)  
   - Use same Gmail OAuth2 credentials.

5. **Create Set Node to Simplify Email:**  
   - Name: "Simplify output before giving to Agent"  
   - Type: Set  
   - Configure to assign a single field `html` with value extracted from the Gmail get node output HTML (`{{$json.html}}`).

6. **Create GPT-5 LM Chat Node:**  
   - Name: "GPT5 as the input to review is heavy"  
   - Type: LangChain LM Chat OpenAI  
   - Set model to `gpt-5`  
   - Connect OpenAI API credentials.

7. **Create LangChain Agent Node:**  
   - Name: "Judge 👩‍⚖️"  
   - Type: LangChain Agent  
   - Paste the detailed system prompt that defines QA rules and decision logic (see detailed prompt above).  
   - Set OnError to continue with error output.  
   - Connect input from GPT-5 node output.  
   - Connect OpenAI API credentials if needed separately.

8. **Create LangChain Output Parser Node:**  
   - Name: "Expected Format"  
   - Type: LangChain Output Parser Structured  
   - Define JSON schema with required fields: status (PASS/BLOCK), confidence (integer), issues_found (array of objects), recommendation (string).  
   - Connect input from Judge node raw output.

9. **Create IF Node for Decision:**  
   - Name: "Judge happy ?"  
   - Type: IF  
   - Condition: Check if `{{$json.output.status}}` equals "PASS" (case sensitive, strict).  
   - Connect input from Expected Format node output.

10. **Create Set Node for Approval:**  
    - Name: "Send approval to parent workflow"  
    - Type: Set  
    - Assign `output.status` to `{{$json.output.status}}` from Judge output.  
    - Connect as True branch from IF node.

11. **Create Gmail Send Node for Manual Review:**  
    - Name: "Ask Human to review Newsletter"  
    - Type: Gmail (Send Email)  
    - Configure:  
      - Recipient: admin email, e.g., emir.belkahia@gmail.com  
      - Subject: "Newsletter is not approved - view email to see why"  
      - Message: JSON stringified AI Judge output (`{{JSON.stringify($json.output, null, 2)}}`)  
      - Operation: sendAndWait with freeText responseType  
    - Connect as False branch from IF node.  
    - Use Gmail OAuth2 credentials.

12. **Create Gmail Send Node for Error Alerts:**  
    - Name: "Inform human that there are errors"  
    - Type: Gmail (Send Email)  
    - Configure:  
      - Recipient: admin email  
      - Subject: "Judge wasn't able to review the newsletter {{ $now }}"  
      - Message: "Judge wasn't able to review the newsletter of the day. Notification time: {{ $now }}. Administrator must review N8N workflow."  
    - Connect error output from Judge node to this node.  
    - Use Gmail OAuth2 credentials.

13. **Connect Workflow:**  
    - Trigger → Send newsletter to LLM Judge → Wait → Get the newsletter → Simplify output → GPT5 LM Chat → Judge → Expected Format → Judge happy ?  
    - True branch: Judge happy ? → Send approval to parent workflow  
    - False branch: Judge happy ? → Ask Human to review Newsletter  
    - Judge node error: → Inform human that there are errors

14. **Add Sticky Notes:**  
    - Add descriptive sticky notes at key sections for documentation as per the original workflow content.

15. **Credentials Setup:**  
    - Configure Gmail OAuth2 credentials for sending and receiving emails.  
    - Configure OpenAI API key with GPT-5 access for LangChain nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow validates actual Gmail-rendered version of newsletter to catch image loading issues and ensure consistent customer experience.                                                                                                                   | Sticky Note11 content                                                                                         |
| Setup requires Gmail account for sending test emails and retrieving rendered emails, OpenAI API key (GPT-5 recommended), and a parent workflow passing newsletter HTML content.                                                                             | Sticky Note11 content                                                                                         |
| Contact for questions or feedback: Emir Belkahia, email: emir.belkahia@gmail.com, LinkedIn: https://www.linkedin.com/in/emirbelkahia/                                                                                                                     | Sticky Note12                                                                                                 |
| Newsletter expected structure diagram available at: ![Newsletter structure](https://gkfvbshnthawbcaugpwp.supabase.co/storage/v1/object/public/n8n-diagrams/2025%20Nov%2011%20-%20Newsletter%20full.png)                                                     | Sticky Note9                                                                                                  |
| AI Judge’s system prompt includes detailed checks on product data completeness, layout integrity, visual rendering, and data logic with strict PASS/BLOCK decision rules.                                                                                  | Judge 👩‍⚖️ node prompt                                                                                         |
| When in doubt, the Judge blocks sending and alerts a human to protect brand reputation and customer experience.                                                                                                                                           | Judge 👩‍⚖️ node prompt                                                                                         |
| For workflow errors or AI failures, an alert email is sent to the admin requiring manual review.                                                                                                                                                            | Sticky Note1, Inform human that there are errors node                                                        |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated n8n workflow. It respects all applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.