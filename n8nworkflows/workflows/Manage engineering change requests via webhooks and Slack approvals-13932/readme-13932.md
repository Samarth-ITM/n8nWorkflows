Manage engineering change requests via webhooks and Slack approvals

https://n8nworkflows.xyz/workflows/manage-engineering-change-requests-via-webhooks-and-slack-approvals-13932


# Manage engineering change requests via webhooks and Slack approvals

# 1. Workflow Overview

This workflow manages **Engineering Change Requests (ECRs)** submitted through an HTTP webhook, classifies the request by impact level, sends an approval request to Slack, pauses for a human decision through an n8n form, and then posts the final approval or rejection back to Slack.

Typical use cases:
- Mechanical drawing change intake from internal tools, forms, or custom apps
- Lightweight engineering approval flows without a full PLM system
- Human-in-the-loop review for design change requests

## 1.1 Input Reception and Validation
The workflow starts with a **POST webhook** that expects a payload containing at least:
- `drawing_no`
- `change_reason`

It immediately validates those fields and returns an HTTP error if they are missing or empty.

## 1.2 ECR Metadata Generation
If validation succeeds, the workflow generates:
- a secure token
- an ECR identifier
- an initial status of `Pending`

These values are appended to the incoming data for downstream processing.

## 1.3 Impact Classification
The workflow classifies the request into an `impact_level` based on keywords found in `change_reason`:
- `Text` → Low
- `Dimension` → Medium
- `Material` → High
- anything else → Medium fallback

## 1.4 Immediate Webhook Response
Before waiting for human approval, the workflow returns a JSON success response to the original webhook caller. This acknowledges receipt and includes the generated metadata.

## 1.5 Slack Notification and Human Approval
After responding to the webhook caller, the workflow posts an approval message to Slack including the ECR details and a **resume form URL**. It then enters a **Wait** state until a reviewer submits an approval form.

## 1.6 Decision Handling
Once the form is submitted, the workflow routes based on the decision:
- `approve` → Slack approval confirmation
- `reject` → Slack rejection notification

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Validation

### Overview
This block receives the incoming change request and verifies that the required fields are present. If the request is invalid, it returns an HTTP 400 response immediately and the workflow stops.

### Nodes Involved
- Webhook - Receive Drawing Change Request
- IF - Validate Required Fields
- Return Response (Error)

### Node Details

#### 1. Webhook - Receive Drawing Change Request
- **Type and role:** `n8n-nodes-base.webhook`  
  Entry point for inbound HTTP requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `drawing-change-request`
  - Response mode: `responseNode`
- **Key expressions or variables used:** None in the node itself, but downstream nodes access values under `{{$json.body...}}`.
- **Input and output connections:**
  - No input; this is the trigger
  - Output → `IF - Validate Required Fields`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases / failures:**
  - Wrong HTTP method produces no valid trigger
  - If upstream sends malformed JSON or unexpected structure, later expressions like `$json.body["drawing_no"]` may fail or evaluate empty
  - Because response mode is `responseNode`, the workflow must eventually hit a Respond to Webhook node for the caller to receive a proper response
- **Sub-workflow reference:** None

#### 2. IF - Validate Required Fields
- **Type and role:** `n8n-nodes-base.if`  
  Validates whether required request fields are missing or empty.
- **Configuration choices:**
  - Uses an `or` condition group
  - Conditions check:
    - `{{$json.body["drawing_no"]}}` is empty
    - `{{$json.body["change_reason"]}}` is empty
    - `{{$json.body["change_reason"].length}} == 0`
- **Key expressions or variables used:**
  - `{{$json.body["drawing_no"]}}`
  - `{{$json.body["change_reason"]}}`
  - `{{$json.body["change_reason"].length}}`
- **Interpretation of logic:**
  - If **any** required field is empty, route to error
  - Otherwise continue processing
- **Input and output connections:**
  - Input ← `Webhook - Receive Drawing Change Request`
  - Output 0 (true) → `Return Response (Error)`
  - Output 1 (false) → `Generate Secure Token`
- **Version-specific requirements:** Type version `2.3`
- **Edge cases / failures:**
  - If `body` is absent entirely, the expressions may evaluate unexpectedly depending on n8n expression handling
  - `.length` on a non-string/null value can become brittle if upstream payload format changes
  - Loose type validation is enabled, which is slightly more forgiving but can hide type issues
- **Sub-workflow reference:** None

#### 3. Return Response (Error)
- **Type and role:** `n8n-nodes-base.respondToWebhook`  
  Sends an HTTP 400 error back to the caller when validation fails.
- **Configuration choices:**
  - Respond with JSON
  - HTTP status code: `400`
  - Static body:
    - `{"error": "drawing_no and change_reason are required fields."}`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `IF - Validate Required Fields`
  - No downstream output used
- **Version-specific requirements:** Type version `1.5`
- **Edge cases / failures:**
  - If workflow pathing never reaches this node in an invalid request, the caller would hang; current wiring is correct
  - Body is static, so it does not identify which exact field failed
- **Sub-workflow reference:** None

---

## Block 2 — ECR Metadata Generation

### Overview
This block enriches the validated request with a generated token, an ECR ID, and an initial status. These fields support tracking and acknowledgment.

### Nodes Involved
- Generate Secure Token
- Generate ECR Metadata (Set Node)

### Node Details

#### 4. Generate Secure Token
- **Type and role:** `n8n-nodes-base.crypto`  
  Generates a random token for the request.
- **Configuration choices:**
  - Action: `generate`
  - Encoding type: `hex`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `IF - Validate Required Fields`
  - Output → `Generate ECR Metadata (Set Node)`
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:**
  - Token length is not explicitly shown here; default generation behavior should be verified in your n8n version
  - If later systems assume a particular token length/format, this should be standardized
- **Sub-workflow reference:** None

#### 5. Generate ECR Metadata (Set Node)
- **Type and role:** `n8n-nodes-base.set`  
  Adds workflow metadata while preserving all existing fields.
- **Configuration choices:**
  - `includeOtherFields: true`
  - Adds:
    - `ecr_id` = `ECR-{{ $now.format('yyyyMMdd-HHmm') }}`
    - `token` = `{{$json.data}}`
    - `status` = `Pending`
- **Key expressions or variables used:**
  - `ECR-{{ $now.format('yyyyMMdd-HHmm') }}`
  - `{{$json.data}}`
- **Interpretation:**
  - The token produced by the Crypto node is expected in `data`
  - ECR IDs are time-based to the minute, not guaranteed unique under concurrency
- **Input and output connections:**
  - Input ← `Generate Secure Token`
  - Output → `Switch - Determine Approver by Impact`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - If the Crypto node output format changes, `{{$json.data}}` may not hold the token
  - ECR ID collisions are possible if multiple requests arrive in the same minute
  - `$now.format(...)` depends on the server timezone unless otherwise controlled
- **Sub-workflow reference:** None

---

## Block 3 — Impact Classification

### Overview
This block assigns an impact level based on keywords found in the change reason. The workflow uses simple text matching rather than a full rules engine.

### Nodes Involved
- Switch - Determine Approver by Impact
- Set Low
- Set Medium
- Set High
- Set Medium (Fallback)

### Node Details

#### 6. Switch - Determine Approver by Impact
- **Type and role:** `n8n-nodes-base.switch`  
  Routes the request into one of several impact categories.
- **Configuration choices:**
  - Rule 1: if `change_reason` contains `Text`
  - Rule 2: if `change_reason` contains `Dimension`
  - Rule 3: if `change_reason` contains `Material`
  - Fallback output enabled as `extra`
- **Key expressions or variables used:**
  - `{{$json.body["change_reason"]}}`
- **Interpretation:**
  - `Text` → first output
  - `Dimension` → second output
  - `Material` → third output
  - no match → fallback fourth output
- **Input and output connections:**
  - Input ← `Generate ECR Metadata (Set Node)`
  - Output 0 → `Set Low`
  - Output 1 → `Set Medium`
  - Output 2 → `Set High`
  - Output 3 → `Set Medium (Fallback)`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - Matching is case-sensitive in practice according to the condition settings
  - Requests containing multiple keywords will route according to switch behavior/order; first matching route behavior should be validated in your instance
  - The node name mentions “Approver” but it actually determines impact, not a person or role
- **Sub-workflow reference:** None

#### 7. Set Low
- **Type and role:** `n8n-nodes-base.set`  
  Assigns `impact_level = Low`.
- **Configuration choices:**
  - `includeOtherFields: true`
  - Adds `impact_level: Low`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Switch - Determine Approver by Impact`
  - Output → `Return Success (Respond to Webhook)`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:** Minimal; depends on upstream payload remaining intact
- **Sub-workflow reference:** None

#### 8. Set Medium
- **Type and role:** `n8n-nodes-base.set`  
  Assigns `impact_level = Medium`.
- **Configuration choices:**
  - `includeOtherFields: true`
  - Adds `impact_level: Medium`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Switch - Determine Approver by Impact`
  - Output → `Return Success (Respond to Webhook)`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:** Minimal
- **Sub-workflow reference:** None

#### 9. Set High
- **Type and role:** `n8n-nodes-base.set`  
  Assigns `impact_level = High`.
- **Configuration choices:**
  - `includeOtherFields: true`
  - Adds `impact_level: High`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Switch - Determine Approver by Impact`
  - Output → `Return Success (Respond to Webhook)`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:** Minimal
- **Sub-workflow reference:** None

#### 10. Set Medium (Fallback)
- **Type and role:** `n8n-nodes-base.set`  
  Assigns a default `impact_level = Medium` when no keyword matches.
- **Configuration choices:**
  - `includeOtherFields: true`
  - Adds `impact_level: Medium`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Switch - Determine Approver by Impact`
  - Output → `Return Success (Respond to Webhook)`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - Fallback logic may over-classify unmatched requests as Medium instead of flagging them for manual review
- **Sub-workflow reference:** None

---

## Block 4 — Immediate Webhook Response

### Overview
This block returns a success acknowledgment to the original caller before the workflow moves into the Slack and human-approval phase.

### Nodes Involved
- Return Success (Respond to Webhook)

### Node Details

#### 11. Return Success (Respond to Webhook)
- **Type and role:** `n8n-nodes-base.respondToWebhook`  
  Returns a JSON acknowledgment to the HTTP client.
- **Configuration choices:**
  - Respond with JSON
  - Response body includes:
    - `message`
    - `id`
    - `token`
    - `change_reason`
    - `drawing_no`
    - `impact_level`
- **Key expressions or variables used:**
  - `{{$json.ecr_id}}`
  - `{{$json.token}}`
  - `{{$json.body.change_reason}}`
  - `{{$json.body.drawing_no}}`
  - `{{$json.impact_level}}`
- **Input and output connections:**
  - Input ← one of:
    - `Set Low`
    - `Set Medium`
    - `Set High`
    - `Set Medium (Fallback)`
  - Output → `Notify Slack (Approval Request)`
- **Version-specific requirements:** Type version `1.5`
- **Edge cases / failures:**
  - If upstream body fields are absent or nested differently, response generation may produce blank values
  - Continuing after `Respond to Webhook` is valid here and is required for the asynchronous approval flow
- **Sub-workflow reference:** None

---

## Block 5 — Slack Notification and Human Approval

### Overview
This block notifies Slack that a new ECR requires review, then pauses execution until a human submits the embedded approval form.

### Nodes Involved
- Notify Slack (Approval Request)
- Wait (Form)

### Node Details

#### 12. Notify Slack (Approval Request)
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a formatted approval request message to a Slack channel.
- **Configuration choices:**
  - Authentication: OAuth2
  - Operation: send message to a selected channel
  - Channel ID currently set as placeholder expression `=Enter your channel ID`
  - Message includes:
    - ECR ID
    - change reason
    - impact level
    - `{{$execution.resumeFormUrl}}`
- **Key expressions or variables used:**
  - `{{$json.ecr_id}}`
  - `{{$json.body.change_reason}}`
  - `{{$json.impact_level}}`
  - `{{$execution.resumeFormUrl}}`
- **Input and output connections:**
  - Input ← `Return Success (Respond to Webhook)`
  - Output → `Wait (Form)`
- **Version-specific requirements:** Type version `2.4`
- **Edge cases / failures:**
  - Requires valid Slack OAuth2 credentials
  - Placeholder channel ID must be replaced
  - If `resumeFormUrl` is unavailable due to execution mode or environment restrictions, reviewers will not get a working approval link
  - Slack permission scopes must allow posting in the chosen channel
- **Sub-workflow reference:** None

#### 13. Wait (Form)
- **Type and role:** `n8n-nodes-base.wait`  
  Suspends workflow execution until a human submits a form.
- **Configuration choices:**
  - Resume mode: `form`
  - Form title: `Engineering Change Request (ECR) Approval`
  - Description: `Please review the details and provide your decision.`
  - Fields:
    - Dropdown `decision` with `approve` and `reject`
    - Textarea `comment`
- **Key expressions or variables used:** None
- **Expected output after resume:**
  - `decision`
  - `comment`
- **Input and output connections:**
  - Input ← `Notify Slack (Approval Request)`
  - Output → `Check Decision`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases / failures:**
  - Requires n8n environment support for wait/resume and public form access as applicable
  - Long waits may be affected by execution retention settings
  - If users never submit the form, execution remains paused indefinitely
  - No validation enforces comment content
- **Sub-workflow reference:** None

---

## Block 6 — Decision Handling

### Overview
This block inspects the human decision from the Wait form and sends the final result to Slack.

### Nodes Involved
- Check Decision
- Notify Approval (Slack)
- Notify Rejection (Slack)

### Node Details

#### 14. Check Decision
- **Type and role:** `n8n-nodes-base.switch`  
  Routes execution based on the submitted approval decision.
- **Configuration choices:**
  - Rule 1: `{{$json.decision}} == approve`
  - Rule 2: `{{$json.decision}} == reject`
- **Key expressions or variables used:**
  - `{{$json.decision}}`
- **Input and output connections:**
  - Input ← `Wait (Form)`
  - Output 0 → `Notify Approval (Slack)`
  - Output 1 → `Notify Rejection (Slack)`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - Any unexpected value outside `approve`/`reject` is not handled
  - No fallback branch is configured, so malformed form data could silently end execution
- **Sub-workflow reference:** None

#### 15. Notify Approval (Slack)
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a Slack message confirming approval.
- **Configuration choices:**
  - Authentication: OAuth2
  - Channel ID placeholder: `=Enter your channel ID`
  - Text: `✅ ECR Approved. Proceeding to ECO issuance.`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Check Decision`
  - No downstream output used
- **Version-specific requirements:** Type version `2.4`
- **Edge cases / failures:**
  - Requires valid Slack credentials and channel access
  - Message is static and does not include ECR ID or reviewer comment
- **Sub-workflow reference:** None

#### 16. Notify Rejection (Slack)
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a Slack message confirming rejection.
- **Configuration choices:**
  - Authentication: OAuth2
  - Channel ID placeholder: `=Enter your channel ID`
  - Text: `❌ ECR Rejected. Please review and resubmit.`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Check Decision`
  - No downstream output used
- **Version-specific requirements:** Type version `2.4`
- **Edge cases / failures:**
  - Requires valid Slack credentials and channel access
  - Message is static and does not include rejection comment
- **Sub-workflow reference:** None

---

## Block 7 — Documentation / Canvas Notes

### Overview
These nodes are visual documentation only. They do not execute but are important for understanding the intended workflow structure and setup guidance.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### 17. Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation containing the overall description, setup guidance, and requirements.
- **Configuration choices:** Large note covering the whole workflow canvas
- **Content summary:**
  - Overview of ECR via Slack approvals
  - Setup steps
  - Requirements
- **Connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None; non-executable
- **Sub-workflow reference:** None

#### 18. Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Label for validation area
- **Content:** `Validate Incoming Request`
- **Connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### 19. Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Label for metadata and impact assignment area
- **Content:** `Determine Impact Level & Assign Metadata`
- **Connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### 20. Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Label for approval request section
- **Content:** `Notify & Wait for Human Approval`
- **Connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### 21. Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Label for decision-processing section
- **Content:** `Process Decision`
- **Connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Receive Drawing Change Request | Webhook | Receives incoming POST requests for ECR submission |  | IF - Validate Required Fields | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications |
| IF - Validate Required Fields | IF | Checks that drawing number and change reason are present | Webhook - Receive Drawing Change Request | Return Response (Error); Generate Secure Token | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Validate Incoming Request |
| Return Response (Error) | Respond to Webhook | Returns 400 error for invalid submissions | IF - Validate Required Fields |  | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Validate Incoming Request |
| Generate Secure Token | Crypto | Generates a random token for tracking or acknowledgment | IF - Validate Required Fields | Generate ECR Metadata (Set Node) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications |
| Generate ECR Metadata (Set Node) | Set | Adds ECR ID, status, and token to the payload | Generate Secure Token | Switch - Determine Approver by Impact | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Determine Impact Level & Assign Metadata |
| Switch - Determine Approver by Impact | Switch | Routes by keyword to assign impact level | Generate ECR Metadata (Set Node) | Set Low; Set Medium; Set High; Set Medium (Fallback) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Determine Impact Level & Assign Metadata |
| Set Low | Set | Sets Low impact classification | Switch - Determine Approver by Impact | Return Success (Respond to Webhook) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Determine Impact Level & Assign Metadata |
| Set Medium | Set | Sets Medium impact classification | Switch - Determine Approver by Impact | Return Success (Respond to Webhook) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Determine Impact Level & Assign Metadata |
| Set High | Set | Sets High impact classification | Switch - Determine Approver by Impact | Return Success (Respond to Webhook) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Determine Impact Level & Assign Metadata |
| Set Medium (Fallback) | Set | Sets default Medium impact when no keyword matches | Switch - Determine Approver by Impact | Return Success (Respond to Webhook) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Determine Impact Level & Assign Metadata |
| Return Success (Respond to Webhook) | Respond to Webhook | Returns acknowledgment to the original webhook caller | Set Low; Set Medium; Set High; Set Medium (Fallback) | Notify Slack (Approval Request) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications |
| Notify Slack (Approval Request) | Slack | Sends approval request into Slack with resume form URL | Return Success (Respond to Webhook) | Wait (Form) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Notify & Wait for Human Approval |
| Wait (Form) | Wait | Pauses execution until reviewer submits decision form | Notify Slack (Approval Request) | Check Decision | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Notify & Wait for Human Approval |
| Check Decision | Switch | Routes based on form decision approve or reject | Wait (Form) | Notify Approval (Slack); Notify Rejection (Slack) | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Process Decision |
| Notify Approval (Slack) | Slack | Sends Slack confirmation for approved ECR | Check Decision |  | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Process Decision |
| Notify Rejection (Slack) | Slack | Sends Slack notification for rejected ECR | Check Decision |  | ## ⚙️ Manage Engineering Change Requests (ECR) via Slack<br>## 📋 Overview<br>Streamline your engineering change management process by integrating generic webhooks with Slack interactive approvals. This workflow receives a generic design change request, categorizes its impact level, and routes it to a Slack channel for approval.<br>This template is ideal for **Mechanical Engineering teams**, **Manufacturing**, and **Project Managers** who need to track drawing changes (ECR/ECO) without complex PLM software.<br>## 🚀 How it works<br>1. **Receive Request:** Catches a POST webhook containing `drawing_no` and `change_reason`.<br>2. **Validation & Logic:** Ensures data integrity and determines the "Impact Level" (High, Medium, Low) based on keywords in the reason (e.g., "Dimension" triggers specific logic).<br>3. **Notification:** Sends a formatted message to a designated Slack channel with a link to an approval form.<br>4. **Human in the Loop:** The workflow waits (pauses) until a reviewer approves or rejects the request via the n8n form.<br>5. **Final Notification:** Posts the final decision back to Slack.<br>## 🛠️ Setup Steps<br>1. **Configure Webhook:** Use the production URL of the Webhook node in your upstream application (e.g., Google Forms, internal tools, or Typeform).<br>2. **Connect Slack:** Select your Slack credential and choose the channel where approval requests should be sent in the **Notify Slack** nodes.<br>3. **Customize Logic:** Open the **Switch** node to adjust the keywords (e.g., "Material", "Dimension") that determine the impact level according to your company's policy.<br>## 💡 Requirements<br>- n8n (Self-hosted or Cloud)<br>- Slack account and a channel for notifications<br>Process Decision |
| Sticky Note | Sticky Note | Visual documentation for workflow overview and setup |  |  |  |
| Sticky Note1 | Sticky Note | Visual label for validation area |  |  |  |
| Sticky Note2 | Sticky Note | Visual label for metadata and impact section |  |  |  |
| Sticky Note3 | Sticky Note | Visual label for approval section |  |  |  |
| Sticky Note4 | Sticky Note | Visual label for decision section |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Mechanical Design Change Approval Workflow`.

2. **Add the Webhook trigger**
   - Add a **Webhook** node.
   - Name it: `Webhook - Receive Drawing Change Request`.
   - Set:
     - **HTTP Method**: `POST`
     - **Path**: `drawing-change-request`
     - **Response Mode**: `Using Respond to Webhook Node` / `responseNode`
   - This node will become the public entry point.

3. **Add request validation**
   - Add an **IF** node.
   - Name it: `IF - Validate Required Fields`.
   - Connect the webhook to this IF node.
   - Configure conditions using **OR** logic:
     - `{{$json.body["drawing_no"]}}` → `is empty`
     - `{{$json.body["change_reason"]}}` → `is empty`
     - `{{$json.body["change_reason"].length}}` → `equals` → `0`
   - Keep type handling permissive if available, similar to loose validation.

4. **Add the error response branch**
   - Add a **Respond to Webhook** node.
   - Name it: `Return Response (Error)`.
   - Connect it to the **true/error** output of the IF node.
   - Set:
     - **Respond With**: `JSON`
     - **Response Code**: `400`
     - **Response Body**:
       ```json
       {"error": "drawing_no and change_reason are required fields."}
       ```

5. **Add secure token generation**
   - Add a **Crypto** node.
   - Name it: `Generate Secure Token`.
   - Connect it to the **false/valid** output of the IF node.
   - Configure:
     - **Action**: `Generate`
     - **Encoding Type**: `Hex`

6. **Add metadata enrichment**
   - Add a **Set** node.
   - Name it: `Generate ECR Metadata (Set Node)`.
   - Connect `Generate Secure Token` to it.
   - Enable **Include Other Input Fields**.
   - Add these fields:
     - `ecr_id` as string:
       - `ECR-{{ $now.format('yyyyMMdd-HHmm') }}`
     - `token` as string:
       - `{{$json.data}}`
     - `status` as string:
       - `Pending`

7. **Add impact classification switch**
   - Add a **Switch** node.
   - Name it: `Switch - Determine Approver by Impact`.
   - Connect `Generate ECR Metadata (Set Node)` to it.
   - Add rule outputs:
     1. `{{$json.body["change_reason"]}}` contains `Text`
     2. `{{$json.body["change_reason"]}}` contains `Dimension`
     3. `{{$json.body["change_reason"]}}` contains `Material`
   - Enable a **fallback/extra output**.

8. **Add the impact assignment nodes**
   - Add four **Set** nodes, all with **Include Other Input Fields** enabled:
     - `Set Low` → set `impact_level = Low`
     - `Set Medium` → set `impact_level = Medium`
     - `Set High` → set `impact_level = High`
     - `Set Medium (Fallback)` → set `impact_level = Medium`
   - Connect:
     - Output 1 of the Switch → `Set Low`
     - Output 2 → `Set Medium`
     - Output 3 → `Set High`
     - Fallback output → `Set Medium (Fallback)`

9. **Add the success webhook response**
   - Add a **Respond to Webhook** node.
   - Name it: `Return Success (Respond to Webhook)`.
   - Connect all four Set nodes into this same node.
   - Set:
     - **Respond With**: `JSON`
     - **Response Body**:
       ```json
       {
         "message": "Request received. Approval process started.",
         "id": "{{ $json.ecr_id }}",
         "token": "{{ $json.token }}",
         "change_reason": "{{ $json.body.change_reason }}",
         "drawing_no": "{{ $json.body.drawing_no }}",
         "impact_level": "{{ $json.impact_level }}"
       }
       ```

10. **Create the Slack approval request node**
    - Add a **Slack** node.
    - Name it: `Notify Slack (Approval Request)`.
    - Connect `Return Success (Respond to Webhook)` to it.
    - Configure Slack OAuth2 credentials.
    - Select message-to-channel mode.
    - Set the target **channel ID** to your real Slack channel.
    - Use this message body:
      ```text
      🚀 *New ECR Approval Request*

      *ID:* {{ $json.ecr_id }}
      *Reason:* {{ $json.body.change_reason }}
      *Impact:* {{ $json.impact_level }}

      👇 *Action Required:*
      {{ $execution.resumeFormUrl }}
      ```
    - Important: the workflow depends on `{{$execution.resumeFormUrl}}`, so the Wait node must be placed downstream and your n8n environment must support resumable executions.

11. **Add the human approval wait step**
    - Add a **Wait** node.
    - Name it: `Wait (Form)`.
    - Connect `Notify Slack (Approval Request)` to it.
    - Configure:
      - **Resume**: `On Form Submission`
      - **Form Title**: `Engineering Change Request (ECR) Approval`
      - **Description**: `Please review the details and provide your decision.`
    - Add form fields:
      - Dropdown field labeled `decision`
        - options: `approve`, `reject`
      - Textarea field labeled `comment`

12. **Add decision routing**
    - Add a **Switch** node.
    - Name it: `Check Decision`.
    - Connect `Wait (Form)` to it.
    - Configure two rules:
      - `{{$json.decision}}` equals `approve`
      - `{{$json.decision}}` equals `reject`

13. **Add approval notification**
    - Add a **Slack** node.
    - Name it: `Notify Approval (Slack)`.
    - Connect the first output of `Check Decision` to it.
    - Configure Slack OAuth2 credentials.
    - Set the channel ID to your target channel.
    - Message:
      - `✅ ECR Approved. Proceeding to ECO issuance.`

14. **Add rejection notification**
    - Add a **Slack** node.
    - Name it: `Notify Rejection (Slack)`.
    - Connect the second output of `Check Decision` to it.
    - Configure Slack OAuth2 credentials.
    - Set the channel ID to your target channel.
    - Message:
      - `❌ ECR Rejected. Please review and resubmit.`

15. **Add optional sticky notes for maintainability**
    - Add sticky notes matching the original layout:
      - `Validate Incoming Request`
      - `Determine Impact Level & Assign Metadata`
      - `Notify & Wait for Human Approval`
      - `Process Decision`
      - A larger overview note with setup instructions and requirements

16. **Configure credentials**
    - **Slack OAuth2**
      - Create or select a Slack OAuth2 credential in n8n
      - Ensure the app has permission to post to the chosen channel
      - Replace all placeholder channel IDs with real values

17. **Activate and test**
    - Save the workflow.
    - Use the webhook test or production URL.
    - Send a sample POST body such as:
      ```json
      {
        "drawing_no": "DWG-1042",
        "change_reason": "Dimension update for mounting hole spacing"
      }
      ```
    - Expected behavior:
      - Caller gets immediate JSON success response
      - Slack receives approval request
      - Reviewer opens form URL, chooses approve/reject, optionally comments
      - Slack receives final decision message

18. **Recommended hardening after reproduction**
    - Make keyword matching case-insensitive
    - Add fallback handling for unexpected decision values
    - Include `ecr_id` and `comment` in final Slack messages
    - Use a more collision-resistant ECR ID than minute-based timestamp alone
    - Add authentication or signature validation on the webhook if exposed publicly

### Sub-workflow setup
This workflow does **not** use any Execute Workflow / sub-workflow nodes. No child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Manage Engineering Change Requests (ECR) via Slack. Streamlines webhook intake, impact classification, Slack notification, wait-for-approval, and final decision posting. | Overall workflow purpose |
| Intended for Mechanical Engineering teams, Manufacturing, and Project Managers who need to track drawing changes without complex PLM software. | Usage context |
| Use the production URL of the Webhook node in upstream systems such as Google Forms, internal tools, or Typeform. | Webhook integration guidance |
| Connect Slack credentials and select the correct notification channel in all Slack nodes. | Slack setup |
| Customize the Switch node keywords such as `Material` and `Dimension` to align with company policy. | Impact classification customization |
| Requirements: n8n (self-hosted or cloud), Slack account, and a Slack channel for notifications. | Environment prerequisites |

## Additional implementation notes
- The workflow has **one entry point**: the webhook.
- It contains **no sub-workflows**.
- The node named **“Switch - Determine Approver by Impact”** does not actually select an approver; it only sets an impact level.
- The generated `token` is returned to the caller but is not used later in the workflow.
- The reviewer `comment` collected in the form is also not used later.
- The workflow is currently **inactive** in the provided JSON (`"active": false`).