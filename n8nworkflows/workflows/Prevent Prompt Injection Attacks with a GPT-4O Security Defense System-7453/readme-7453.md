Prevent Prompt Injection Attacks with a GPT-4O Security Defense System

https://n8nworkflows.xyz/workflows/prevent-prompt-injection-attacks-with-a-gpt-4o-security-defense-system-7453


# Prevent Prompt Injection Attacks with a GPT-4O Security Defense System

---

### 1. Workflow Overview

This workflow implements a **multi-layered AI security pipeline** designed to detect, assess, sanitize, and safely deliver textual input while preventing prompt injection attacks and other malicious content threats. It targets use cases where user-generated content or external inputs are processed by AI models, requiring stringent security checks to avoid exploitation through prompt injections, code execution attempts, or data exfiltration vectors.

The workflow is logically structured into the following key blocks:

- **1.1 Input Reception and Initial Extraction:** Receives incoming HTTP POST requests and extracts raw input data for processing.
- **1.2 Content Policy Violation Detection:** Uses OpenAI moderation to detect immediate policy violations such as hate speech or sexual content.
- **1.3 Input Validation & Threat Assessment:** A custom GPT-4O agent performs deep threat detection including prompt injection, code injection, and credential harvesting.
- **1.4 Content Sanitization & Neutralization:** Another GPT-4O agent sanitizes and neutralizes detected threats while preserving legitimate content.
- **1.5 Content Formatting and Presentation Optimization:** Formats sanitized content for optimized delivery across platforms.
- **1.6 Final Quality Assurance & Delivery Readiness:** Performs final quality checks and decides delivery readiness.
- **1.7 Decision Routing & Response:** Routes the output based on validation results, sends rejection emails if needed, and responds to the original webhook request.

Each block builds upon the previous one, applying increasingly refined AI-driven security and content processing techniques to ensure both safety and usability.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initial Extraction

- **Overview:** Captures incoming POST HTTP requests on a webhook, extracts the message payload for downstream processing.
- **Nodes Involved:** `Webhook`, `Extract Data`
- **Node Details:**

  - **Webhook**
    - Type: HTTP Webhook (n8n built-in)
    - Role: Entry point endpoint listening on path `/sanity-check` with POST method.
    - Configuration: Responds immediately through response node; does not ignore bots.
    - Inputs: External HTTP POST requests.
    - Outputs: Raw request JSON passed downstream.
    - Edge Cases: Invalid HTTP method, missing payload, malformed JSON.
  
  - **Extract Data**
    - Type: Set Node
    - Role: Extracts the property `body` from webhook JSON and assigns it to `data` for clarity.
    - Configuration: Assigns `data` as the entire body object.
    - Inputs: Raw webhook JSON.
    - Outputs: Object with `data` field containing user input.
    - Edge Cases: Empty body or unexpected payload structure.

---

#### 1.2 Content Policy Violation Detection (Layer-1)

- **Overview:** Uses OpenAI’s moderation/classification API to detect content violating policies such as sexual, hate, harassment, self-harm, or violence categories. Immediate stop if violations found.
- **Nodes Involved:** `Text violations`, `Validate trueCategories`, `Check Success`, `Merge`
- **Node Details:**

  - **Text violations**
    - Type: OpenAI (LangChain node)
    - Role: Content classification to flag policy violations.
    - Configuration: Operation set to `classify` with default options.
    - Inputs: Raw user message extracted from `data.message`.
    - Outputs: Categories flagged true/false.
    - Credentials: OpenAI API key.
    - Edge Cases: API rate limit, incomplete classification, ambiguous content.

  - **Validate trueCategories**
    - Type: Code Node (JavaScript)
    - Role: Filters categories flagged true among a fixed list of sensitive categories.
    - Configuration: Checks categories such as sexual, hate, harassment, self-harm, violence, etc.
    - Outputs: Status `failure` if any violation found; otherwise `success`.
    - Edge Cases: Missing category data, unexpected category structure.

  - **Check Success**
    - Type: If Node
    - Role: Routes flow based on status from `Validate trueCategories`.
    - Configuration: Passes only if `status == "success"`.
    - Outputs: Continues pipeline or responds immediately if failure.
  
  - **Merge**
    - Type: Merge Node
    - Role: Combines outputs from `Extract Data` and `Check Success` for synchronized downstream flow.

---

#### 1.3 Input Validation & Threat Assessment (Layer-2)

- **Overview:** Performs deep analysis for prompt injection, code execution, credential harvesting, encoding tricks, and other advanced threat vectors. Determines if input should be rejected outright or processed further.
- **Nodes Involved:** `Input Validation & Pattern Detection`, `Is REJECTED?`
- **Node Details:**

  - **Input Validation & Pattern Detection**
    - Type: OpenAI (LangChain node)
    - Role: GPT-4O agent specialized in threat detection and classification.
    - Configuration:
      - Model: GPT-4O
      - Temperature: 0.1, TopP: 0.7 (low randomness for reliability)
      - System prompt includes detailed threat patterns and decision matrix.
      - Input: The original user input message.
      - Output: JSON with fields including `status` (e.g., REJECTED, CLEAN), threat classification, and assessment.
    - Credentials: OpenAI API.
    - Edge Cases: API timeout, malformed input, unexpected JSON output.

  - **Is REJECTED?**
    - Type: If Node
    - Role: Branches flow based on whether input was flagged REJECTED.
    - Configuration: Checks if `message.content.status` equals `REJECTED`.
    - Outputs: 
      - YES: Triggers rejection reporting and stops further processing.
      - NO: Continues to sanitization layer.

---

#### 1.4 Content Sanitization & Neutralization (Layer-3)

- **Overview:** Cleans and neutralizes detected threats based on validation results, removing malicious code, URLs, injection attempts, and encoding attacks while preserving legitimate content.
- **Nodes Involved:** `Content Sanitization & Neutralization`, `REJECTED?`
- **Node Details:**

  - **Content Sanitization & Neutralization**
    - Type: OpenAI (LangChain node)
    - Role: GPT-4O agent performing comprehensive sanitization protocols.
    - Configuration:
      - Model: GPT-4O
      - Temperature: 0.1, TopP: 0.7
      - System prompt instructs aggressive threat neutralization and preservation of educational/legitimate content.
    - Inputs: Validated input and threat assessment results.
    - Outputs: Sanitized content and detailed sanitization report.
    - Credentials: OpenAI API.
    - Edge Cases: Over-sanitization, incomplete threat removal.

  - **REJECTED?**
    - Type: If Node
    - Role: Checks if sanitization step resulted in status `REJECTED` (e.g., new threats found).
    - Configuration: Conditional routing based on sanitized content status.
    - Outputs: 
      - YES: Respond with rejection.
      - NO: Proceed to formatting.

---

#### 1.5 Content Formatting and Presentation Optimization (Layer-4)

- **Overview:** Formats sanitized content for platform-appropriate presentation, improving readability and accessibility without altering content security.
- **Nodes Involved:** `Format Content`
- **Node Details:**

  - **Format Content**
    - Type: OpenAI (LangChain node)
    - Role: Final output encoding and contextualization for presentation.
    - Configuration:
      - Model: GPT-4O
      - Temperature: 0.1, TopP: 0.7
      - System prompt mandates preservation of input string, applying only formatting.
    - Input: Sanitized content.
    - Output: Presentation-ready content in markdown or JSON.
    - Credentials: OpenAI API.
    - Edge Cases: Formatting errors, loss of readability.

---

#### 1.6 Final Quality Assurance & Delivery Readiness (Layer-5)

- **Overview:** Performs final quality control, checking that sanitization was complete, formatting intact, and content suitable for delivery to end user.
- **Nodes Involved:** `Final Quality Assurance & Delivery Readiness`, `Merge1`, `Edit Fields`, `Switch`
- **Node Details:**

  - **Final Quality Assurance & Delivery Readiness**
    - Type: OpenAI (LangChain node)
    - Role: Quality inspector validating pipeline output.
    - Configuration:
      - Model: GPT-4O
      - Checks content integrity, residual anomalies, delivery readiness.
      - Outputs decision: DELIVER, REPROCESS, or ESCALATE_REVIEW.
    - Credentials: OpenAI API.
    - Edge Cases: False negatives or positives on quality issues.

  - **Merge1**
    - Type: Merge Node
    - Role: Combines multiple input streams for final output preparation.

  - **Edit Fields**
    - Type: Set Node
    - Role: Maps final output fields into unified JSON structure for response and logging.

  - **Switch**
    - Type: Switch Node
    - Role: Routes workflow based on final status:
      - DELIVER: Send to webhook response.
      - ESCALATE_REVIEW: Loop back to validation for manual review.
      - REPROCESS: Routes back to input validation for another pass.

---

#### 1.7 Decision Routing & Response

- **Overview:** Handles output responses including sending rejection reports via email and responding to the webhook with either sanitized content or rejection messages.
- **Nodes Involved:** `Report Rejection`, `EMAIL`, `Custom Message`, `Respond to Webhook`
- **Node Details:**

  - **Report Rejection**
    - Type: Set Node
    - Role: Prepares detailed rejection report data for email notification.
    - Inputs: Rejection details from validation nodes.
    - Outputs: Structured JSON for email content.

  - **EMAIL**
    - Type: Email Send Node
    - Role: Sends email notification to admin about detected prompt injection attacks.
    - Configuration:
      - SMTP credentials required.
      - Email body includes IP, headers, AI report, threat assessment.
      - Subject line set dynamically from rejection reason.
    - Edge Cases: SMTP auth failure, email delivery failure.

  - **Custom Message**
    - Type: Set Node
    - Role: Sets a default user-facing message for rejected inputs.
    - Output: Static text "Unable to process your request at this time. Please try again later."

  - **Respond to Webhook**
    - Type: Respond to Webhook Node
    - Role: Sends the final response back to the original webhook caller.
    - Inputs:
      - For accepted content: Full processed message.
      - For rejected input: Custom message or rejection notice.
    - Edge Cases: Response timeout, malformed response.

---

### 3. Summary Table

| Node Name                          | Node Type                      | Functional Role                                         | Input Node(s)                          | Output Node(s)                      | Sticky Note                                                                                                                                                   |
|----------------------------------|-------------------------------|--------------------------------------------------------|--------------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | HTTP Webhook                  | Entry point for incoming POST requests                 | -                                    | Extract Data                      |                                                                                                                                                               |
| Extract Data                     | Set                          | Extracts raw input payload from webhook body            | Webhook                              | Text violations, Merge            |                                                                                                                                                               |
| Text violations                  | OpenAI (LangChain)            | Content policy violation detection (OpenAI moderation) | Extract Data                        | Validate trueCategories            | 🚨 **Text Violations**: Detects policy violations and stops workflow if any violation found.                                                                   |
| Validate trueCategories          | Code                         | Filters true categories and sets failure/success status| Text violations                    | Check Success                    | Layer-1                                                                                                                                                       |
| Check Success                   | If                           | Branches flow based on violation status                 | Validate trueCategories              | Merge, Respond to Webhook          |                                                                                                                                                               |
| Merge                           | Merge                        | Combines data and check success results                  | Extract Data, Check Success          | Input Validation & Pattern Detection |                                                                                                                                                               |
| Input Validation & Pattern Detection | OpenAI (LangChain)            | Deep threat detection and threat classification         | Merge                              | Is REJECTED?                     | 🛡️ **Input Validation & Pattern Detection**: Critical gatekeeper for prompt injection and threat detection.                                                   |
| Is REJECTED?                    | If                           | Branches flow based on rejection status                  | Input Validation & Pattern Detection | Report Rejection, Content Sanitization & Neutralization |                                                                                                                                                               |
| Report Rejection                | Set                          | Prepares rejection report data for email notification   | Is REJECTED? (Yes branch)            | EMAIL                           |                                                                                                                                                               |
| EMAIL                          | Email Send                   | Sends email notification on detected prompt injection   | Report Rejection                    | Custom Message                   |                                                                                                                                                               |
| Custom Message                 | Set                          | Sets default rejection response message                  | EMAIL                             | Respond to Webhook               | Custom message editable for rejected inputs.                                                                                                                 |
| Content Sanitization & Neutralization | OpenAI (LangChain)            | Sanitizes and neutralizes threats in content            | Is REJECTED? (No branch)             | REJECTED?                      | 🧼 **Content Sanitization & Neutralization**: Removes threats while preserving legitimate content.                                                             |
| REJECTED?                      | If                           | Checks if sanitization resulted in rejection             | Content Sanitization & Neutralization | Respond to Webhook, Format Content |                                                                                                                                                               |
| Format Content                 | OpenAI (LangChain)            | Formats sanitized content for optimal presentation       | REJECTED? (No branch)                | Final Quality Assurance & Delivery Readiness | 🎨 **Format Content**: Presentation optimizer for sanitized content.                                                                                           |
| Final Quality Assurance & Delivery Readiness | OpenAI (LangChain)            | Final quality check and delivery readiness               | Format Content                    | Merge1                         | ✅ **Final Quality Assurance & Delivery Readiness**: Quality control before delivery.                                                                         |
| Merge1                         | Merge                        | Combines multiple result streams                          | Report Rejection, Final QA, Format Content, Content Sanitization | Edit Fields                    |                                                                                                                                                               |
| Edit Fields                   | Set                          | Maps final output fields for response                     | Merge1                            | Switch                        |                                                                                                                                                               |
| Switch                        | Switch                       | Routes output based on final status (DELIVER, REPROCESS, ESCALATE_REVIEW) | Edit Fields                      | Respond to Webhook, Input Validation & Pattern Detection |                                                                                                                                                               |
| Respond to Webhook            | Respond to Webhook           | Sends final response back to caller                      | Switch, Check Success, REJECTED?, Custom Message | -                             |                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: HTTP Webhook
   - Path: `sanity-check`
   - Method: POST
   - Response Mode: Respond via Response Node
   - Purpose: Entry point for incoming requests.

2. **Create Set Node ("Extract Data")**
   - Purpose: Extract `body` property from webhook payload.
   - Assign `data` = `{{$json["body"]}}`
   - Connect Webhook → Extract Data.

3. **Create OpenAI Node ("Text violations")**
   - Operation: `classify`
   - Input: `{{$json.data.message}}`
   - Credentials: OpenAI API
   - Connect Extract Data → Text violations.

4. **Create Code Node ("Validate trueCategories")**
   - Paste JavaScript to filter true categories from predefined list.
   - Input: `categories` object from Text violations output.
   - Output: JSON with `status` and `violations`.
   - Connect Text violations → Validate trueCategories.

5. **Create If Node ("Check Success")**
   - Condition: `{{$json.status}} == 'success'`
   - Connect Validate trueCategories → Check Success.

6. **Create Merge Node ("Merge")**
   - Mode: Combine by position
   - Inputs: Extract Data output and Check Success output
   - Connect Extract Data & Check Success → Merge.

7. **Create OpenAI Node ("Input Validation & Pattern Detection")**
   - Model: GPT-4O
   - Temperature: 0.1, TopP: 0.7
   - System prompt: Use provided detailed threat detection instructions.
   - Input: User message from merged data.
   - Credentials: OpenAI API
   - Connect Merge → Input Validation & Pattern Detection.

8. **Create If Node ("Is REJECTED?")**
   - Condition: `{{$json.message.content.status}} == 'REJECTED'`
   - Connect Input Validation & Pattern Detection → Is REJECTED?

9. **Create Set Node ("Report Rejection")**
   - Assign extracted threat assessment, classification, validation flags, findings, and status from rejection.
   - Connect Is REJECTED? (Yes) → Report Rejection.

10. **Create Email Send Node ("EMAIL")**
    - Use SMTP credentials.
    - Subject: `{{$json.message.input.threat.assessment.rejection_reason}}`
    - To/From: set dynamically or fixed.
    - HTML/Text body: Include IP, headers, AI report, threat data as per template.
    - Connect Report Rejection → EMAIL.

11. **Create Set Node ("Custom Message")**
    - Assign static message: "Unable to process your request at this time. Please try again later."
    - Connect EMAIL → Custom Message.

12. **Create Respond to Webhook Node ("Respond to Webhook")**
    - Respond with all incoming items.
    - Connect Custom Message → Respond to Webhook.
    - Also connect Check Success (failure branch) → Respond to Webhook for immediate stops.

13. **Create OpenAI Node ("Content Sanitization & Neutralization")**
    - Model: GPT-4O
    - Temperature: 0.1, TopP: 0.7
    - System prompt: Sanitization protocols with threat intelligence.
    - Input: Output from Input Validation & Pattern Detection (No branch).
    - Credentials: OpenAI API
    - Connect Is REJECTED? (No) → Content Sanitization & Neutralization.

14. **Create If Node ("REJECTED?")**
    - Condition: `{{$json.message.content.status}} == 'REJECTED'`
    - Connect Content Sanitization & Neutralization → REJECTED?

15. **Connect REJECTED? Yes → Respond to Webhook** (stop flow with rejection)
    - No branch → Connect to next node.

16. **Create OpenAI Node ("Format Content")**
    - Model: GPT-4O
    - Temperature: 0.1, TopP: 0.7
    - System prompt: Format sanitized content for presentation only.
    - Input: Sanitized content from previous node.
    - Credentials: OpenAI API
    - Connect REJECTED? No → Format Content.

17. **Create OpenAI Node ("Final Quality Assurance & Delivery Readiness")**
    - Model: GPT-4O
    - Temperature: 0.1, TopP: 0.7
    - System prompt: Final validation and delivery readiness checks.
    - Input: Output from Format Content.
    - Credentials: OpenAI API
    - Connect Format Content → Final Quality Assurance & Delivery Readiness.

18. **Create Merge Node ("Merge1")**
    - Mode: Combine by position
    - Number Inputs: 4 (to gather multiple data streams).
    - Connect Report Rejection, Final QA, Format Content, Content Sanitization → Merge1.

19. **Create Set Node ("Edit Fields")**
    - Map all needed final fields: status, sanitized content, threat assessment, validation flags, critical findings, sanitization actions, quality metrics, delivery notes, etc.
    - Connect Merge1 → Edit Fields.

20. **Create Switch Node ("Switch")**
    - Rules based on `message.content.status`:
      - DELIVER → Respond to Webhook
      - ESCALATE_REVIEW → Input Validation & Pattern Detection (reprocess)
      - REPROCESS → Input Validation & Pattern Detection (reprocess)
    - Connect Edit Fields → Switch.

21. **Connect Respond to Webhook Nodes**
    - Connect Switch DELIVER branch → Respond to Webhook.
    - Connect Check Success failure branch → Respond to Webhook.
    - Connect REJECTED? yes branch → Respond to Webhook.

22. **Add Sticky Notes for Documentation**
    - Include summaries and guidance as per original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow implements a **5-layer defense system** using GPT-4O for security validation, sanitization, formatting, and final QA. It uses early rejection to stop dangerous inputs efficiently.                              | Sticky Note near Webhook and Input Validation nodes.                                              |
| OpenAI’s moderation API is leveraged for initial text violation detection including hate, harassment, self-harm, and sexual content categories.                                                                               | Sticky Note near "Text violations" node.                                                         |
| The Input Validation node applies detailed threat detection for prompt injection, code execution, and credential harvesting using a strict security instruction prompt.                                                      | Sticky Note near "Input Validation & Pattern Detection".                                         |
| Content Sanitization removes malicious code, dangerous URLs, and injection attempts, while preserving educational and legitimate content structure.                                                                           | Sticky Note near "Content Sanitization & Neutralization".                                        |
| The final formatting node prepares content for various delivery platforms, ensuring optimal readability and accessibility without compromising security.                                                                      | Sticky Note near "Format Content".                                                               |
| The workflow sends immediate email alerts upon detecting critical prompt injection attempts, including IP and header info for audit and response.                                                                             | Email node "EMAIL" configured with SMTP; customize email recipient and sender addresses.          |
| The workflow uses a decision matrix to route content for delivery, reprocessing, or escalation based on AI quality assessment results.                                                                                        | Switch node after final editing and QA.                                                          |
| Replace the EMAIL node if desired with other email nodes (e.g., Gmail) and ensure SMTP or OAuth2 credentials are configured properly for email notifications.                                                                  | Sticky Note near Input Validation block.                                                         |
| The entire pipeline assumes availability of OpenAI API credentials with sufficient quota and GPT-4O model access.                                                                                                            | Credentials required for multiple OpenAI nodes.                                                  |
| The system is designed with defensive security in mind, preferring false positives (rejecting suspicious content) over false negatives to prevent breaches.                                                                  | Security instructions embedded in GPT-4O system prompts.                                         |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively from an automated n8n workflow designed with strict adherence to content policies. No illegal, offensive, or protected content is included. All processed data is legal and publicly accessible.

---