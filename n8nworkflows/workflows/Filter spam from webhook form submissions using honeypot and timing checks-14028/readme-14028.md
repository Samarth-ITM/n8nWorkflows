Filter spam from webhook form submissions using honeypot and timing checks

https://n8nworkflows.xyz/workflows/filter-spam-from-webhook-form-submissions-using-honeypot-and-timing-checks-14028


# Filter spam from webhook form submissions using honeypot and timing checks

# 1. Workflow Overview

This workflow implements a lightweight anti-spam backend for website form submissions received through an n8n webhook. It is designed to block common automated spam without using CAPTCHAs, relying instead on three checks:

- a hidden honeypot field
- a submission timing check
- a disposable-email-domain blocklist

If a submission is classified as spam, the workflow still returns a normal `200 OK` response with a generic success message so bots do not detect rejection and retry. If the submission is legitimate, the workflow returns a success response containing cleaned form data, ready for downstream processing.

## 1.1 Input Reception

The workflow starts with a webhook that accepts `POST` requests containing form data in JSON format. The frontend is expected to include both a hidden honeypot field and a hidden timestamp field.

## 1.2 Spam Rule Configuration

A Set node defines all configurable detection parameters, including the honeypot field name, timestamp field name, email field name, minimum allowed submission time, and a list of disposable email domains.

## 1.3 Spam Detection Logic

A Code node reads the webhook payload and applies the three anti-spam checks. It produces a normalized output object with:

- `isSpam`
- `reasons`
- `formData`

The `formData` object excludes technical anti-spam fields.

## 1.4 Routing and Response

An IF node branches depending on the `isSpam` boolean. Spam submissions are acknowledged silently. Legitimate submissions receive a success response containing cleaned data. This is also the natural extension point for email, Slack, CRM, or database integrations.

---

# 2. Block-by-Block Analysis

## Block 1 — Documentation and Frontend Setup Notes

### Overview
This block consists of sticky notes that document the workflow purpose, expected frontend payload, spam-rule customization, detection logic, and response behavior. These nodes do not affect execution but are important for maintenance and reproduction.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Describes the workflow as a spam filter backend for website forms, explains target users, processing logic, setup, and customization guidance.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None beyond standard sticky note support.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Shows the expected POST JSON payload:
  - `name`
  - `email`
  - `message`
  - `website_url`
  - `_timestamp`
  
  Also includes frontend HTML for:
  - hidden honeypot input
  - hidden timestamp field populated on page load
- **Key expressions or variables used:** References `website_url` and `_timestamp`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Explains which parameters to edit in the spam-rule configuration node:
  - `honeypotFieldName`
  - `timestampFieldName`
  - `emailFieldName`
  - `minSubmissionTimeSeconds`
  - `disposableDomains`
- **Key expressions or variables used:** Same names as the Set node output.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Describes the three spam checks and the expected output object structure.
- **Key expressions or variables used:** `isSpam`, `reasons[]`, `formData`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Explains branch behavior:
  - spam branch returns generic success
  - legit branch returns success plus cleaned form data
  - downstream integrations should be added after the IF node on the legit path
- **Key expressions or variables used:** None directly.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Explains why the honeypot HTML implementation works:
  - off-screen absolute positioning
  - `aria-hidden`
  - `tabindex="-1"`
  - avoiding `display:none`
- **Key expressions or variables used:** Mentions configurable field names.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## Block 2 — Input Reception

### Overview
This block receives incoming form submissions through an n8n webhook. It expects a `POST` request and defers the HTTP response until a downstream Respond to Webhook node is reached.

### Nodes Involved
- Receive Form Submission

### Node Details

#### Receive Form Submission
- **Type and technical role:** `n8n-nodes-base.webhook`; HTTP entry point for the workflow.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `form-submit`
  - Response mode: `responseNode`
  
  This means the webhook does not respond immediately and waits for one of the Respond to Webhook nodes.
- **Key expressions or variables used:** Downstream code reads `$('Receive Form Submission').first().json.body`.
- **Input and output connections:**  
  - **Input:** none; this is the trigger node.
  - **Output:** Configure Spam Rules
- **Version-specific requirements:** Uses webhook node type version `2.1`.
- **Edge cases or potential failure types:**
  - If the caller sends data in an unexpected format, `json.body` may be missing or malformed.
  - If no Respond to Webhook node is reached, the request may time out.
  - If the workflow is inactive in production, the webhook endpoint will not process submissions.
  - Payload structure depends on how the frontend sends JSON; incorrect `Content-Type` or non-JSON body may alter the received structure.
- **Sub-workflow reference:** None.

---

## Block 3 — Spam Rule Configuration

### Overview
This block centralizes anti-spam settings so that the detection logic remains configurable without editing JavaScript logic. It produces a single JSON object consumed by the next Code node.

### Nodes Involved
- Configure Spam Rules

### Node Details

#### Configure Spam Rules
- **Type and technical role:** `n8n-nodes-base.set`; emits a configuration object.
- **Configuration choices:**
  - Mode: raw JSON output
  - Produces:
    - `honeypotFieldName: "website_url"`
    - `timestampFieldName: "_timestamp"`
    - `emailFieldName: "email"`
    - `minSubmissionTimeSeconds: 2`
    - `disposableDomains: [...]`
- **Key expressions or variables used:** Output is later read as `$('Configure Spam Rules').first().json`.
- **Input and output connections:**
  - **Input:** Receive Form Submission
  - **Output:** Detect Spam
- **Version-specific requirements:** Uses Set node type version `3.4`; raw JSON mode should be available.
- **Edge cases or potential failure types:**
  - Invalid JSON formatting in `jsonOutput` will break node execution.
  - Mismatched field names versus frontend payload will reduce or disable protection.
  - Domain list entries should be lowercase to match the code’s lowercase normalization.
  - Too aggressive a timing threshold may flag real users.
- **Sub-workflow reference:** None.

---

## Block 4 — Spam Detection Logic

### Overview
This block applies the actual classification logic. It reads the incoming form body and configuration values, evaluates the honeypot, timing, and disposable email checks, and returns a simplified object for routing.

### Nodes Involved
- Detect Spam

### Node Details

#### Detect Spam
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript classification logic.
- **Configuration choices:**
  - Pulls form payload from the webhook node:
    - `$('Receive Form Submission').first().json.body`
  - Pulls config from the Set node:
    - `$('Configure Spam Rules').first().json`
  - Returns one item containing:
    - `isSpam` boolean
    - `reasons` array
    - `formData` object without honeypot and timestamp fields
- **Key expressions or variables used:**
  - `input`
  - `config`
  - `honeypotField`
  - `timestampField`
  - `emailField`
  - `minTime`
  - `disposableDomains`
  - `reasons`
  - `isSpam`
  - `formData`
- **Input and output connections:**
  - **Input:** Configure Spam Rules
  - **Output:** Is Spam?
- **Version-specific requirements:** Uses Code node type version `2`.
- **Edge cases or potential failure types:**
  - If `json.body` is missing, the node returns:
    - `isSpam: false`
    - `reasons: ['No form data received']`
    - `formData: {}`
    
    This is functionally permissive and may not be ideal for strict validation.
  - Invalid timestamps create a `Date` object that may result in `NaN` comparisons; in that case the timing rule simply does not trigger.
  - Future timestamps are ignored by the timing rule because only `diffSeconds >= 0` is checked.
  - Email validation is minimal; malformed emails without `@` simply bypass disposable-domain detection.
  - Disposable-domain matching is exact; subdomains or variant domains are not blocked unless explicitly listed.
  - The node assumes the request body is a flat object.
- **Sub-workflow reference:** None.

**Detection logic implemented**
1. **Honeypot check**
   - If the configured honeypot field exists and is not empty after trimming, mark as spam.

2. **Timing check**
   - If the configured timestamp field exists:
     - parse it as a date
     - compute elapsed seconds from timestamp to current server time
     - if elapsed time is non-negative and less than `minSubmissionTimeSeconds`, mark as spam

3. **Disposable email check**
   - If the configured email field exists:
     - split on `@`
     - lowercase the domain
     - if it exactly matches an entry in `disposableDomains`, mark as spam

4. **Form data cleaning**
   - Removes the honeypot field and timestamp field from the returned `formData`
   - Keeps all other submitted fields untouched

---

## Block 5 — Routing and Response

### Overview
This block evaluates the classification output and returns an HTTP response through one of two response nodes. The spam branch intentionally behaves like a successful submission to avoid signaling rejection to bots.

### Nodes Involved
- Is Spam?
- Silent OK (Spam Blocked)
- Forward & Respond (Legit)

### Node Details

#### Is Spam?
- **Type and technical role:** `n8n-nodes-base.if`; routes items based on `isSpam`.
- **Configuration choices:**
  - Condition checks whether `{{ $json.isSpam }}` equals boolean `true`
  - Strict type validation is enabled
  - Case sensitivity is irrelevant here because the compared value is boolean
- **Key expressions or variables used:**
  - `={{ $json.isSpam }}`
- **Input and output connections:**
  - **Input:** Detect Spam
  - **Output 0 / true branch:** Silent OK (Spam Blocked)
  - **Output 1 / false branch:** Forward & Respond (Legit)
- **Version-specific requirements:** Uses IF node type version `2.3`.
- **Edge cases or potential failure types:**
  - If `isSpam` is missing or not a boolean, strict type validation may route unexpectedly or fail condition evaluation.
  - Because the Code node always returns `isSpam`, this risk is low unless that node is modified.
- **Sub-workflow reference:** None.

#### Silent OK (Spam Blocked)
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; final HTTP response for spam.
- **Configuration choices:**
  - Responds with JSON
  - Adds header `Content-Type: application/json`
  - Response body:
    ```json
    {
      "success": true,
      "message": "Thank you for your submission."
    }
    ```
- **Key expressions or variables used:** Static body; no dynamic expressions.
- **Input and output connections:**
  - **Input:** Is Spam? true branch
  - **Output:** none
- **Version-specific requirements:** Uses Respond to Webhook type version `1.5`.
- **Edge cases or potential failure types:**
  - Must only be used when the webhook node is configured with `responseNode`.
  - If this node is removed and no other response is sent on the spam branch, requests may hang or time out.
- **Sub-workflow reference:** None.

#### Forward & Respond (Legit)
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; final HTTP response for legitimate submissions.
- **Configuration choices:**
  - Responds with JSON
  - Adds header `Content-Type: application/json`
  - Uses an expression to serialize:
    - `success: true`
    - `message: 'Submission received.'`
    - `data: $json.formData`
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ success: true, message: 'Submission received.', data: $json.formData }) }}`
- **Input and output connections:**
  - **Input:** Is Spam? false branch
  - **Output:** none
- **Version-specific requirements:** Uses Respond to Webhook type version `1.5`.
- **Edge cases or potential failure types:**
  - Same `responseNode` dependency as the spam response node.
  - If additional nodes are inserted before this response on the legit branch, ensure a response is still always returned.
  - If downstream modifications remove `formData`, the response may include `undefined` or incomplete data.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Overall workflow documentation and setup notes |  |  | ## Filter Spam from Webhook Form Submissions<br><br>This workflow acts as a **spam filter backend** for your website contact forms. It receives form submissions via webhook, runs three automated checks, and either silently blocks spam or forwards legitimate submissions for processing.<br><br>### Who is this for?<br>Website owners, agencies, or developers who receive form submissions and want to block bots and spam **without CAPTCHAs**, keeping the user experience clean.<br><br>### How it works<br>1. Your frontend sends a POST request with form data, a hidden honeypot field, and a client-side timestamp<br>2. The workflow runs three spam checks:<br>- **Honeypot detection**: If the hidden field contains data, it's a bot<br>- **Timing analysis**: If the form was submitted in under 2 seconds, it's a bot<br>- **Disposable email detection**: Checks the email domain against a configurable blocklist<br>3. **Spam**: Returns a silent 200 OK (the bot thinks it worked, but nothing is forwarded)<br>4. **Legitimate**: Returns success with the cleaned form data for downstream processing<br><br>### Setup<br>1. Activate the workflow<br>2. Add the honeypot field and timestamp to your frontend form (see Step 1 sticky note)<br>3. Optionally adjust the spam rules in the **Configure Spam Rules** node<br><br>### How to customize<br>- Add your own disposable email domains to the blocklist<br>- Adjust the minimum submission time threshold<br>- Connect email, Slack, or CRM nodes after the **Legit** branch to forward real submissions<br><br>**Author:** Florian Eiche, [eiche-digital.de](https://eiche-digital.de) |
| Sticky Note1 | Sticky Note | Documents expected request payload and frontend implementation |  |  | ### Step 1: Receive Form Data<br>POST request with JSON body:<br><pre><code>{<br>  "name": "Max Mustermann",<br>  "email": "max@example.com",<br>  "message": "Hello!",<br>  "website_url": "",<br>  "_timestamp": "2026-03-13T10:00:00Z"<br>}</code></pre><br>**Frontend HTML example:**<br><pre><code>&lt;!-- Honeypot (hidden from users, bots fill it) --&gt;<br>&lt;div style="position:absolute;left:-9999px;"<br>     aria-hidden="true"&gt;<br>  &lt;input type="text" name="website_url"<br>         tabindex="-1" autocomplete="off"&gt;<br>&lt;/div&gt;<br><br>&lt;!-- Timestamp (set on page load) --&gt;<br>&lt;input type="hidden" name="_timestamp" id="ts"&gt;<br>&lt;script&gt;<br>  document.getElementById('ts').value<br>    = new Date().toISOString();<br>&lt;/script&gt;</code></pre> |
| Sticky Note2 | Sticky Note | Documents configurable spam rule parameters |  |  | ### Step 2: Configure Spam Rules<br>Edit the **Configure Spam Rules** node to match your form:<br>- `honeypotFieldName`: name of the hidden honeypot field<br>- `timestampFieldName`: name of the hidden timestamp field<br>- `emailFieldName`: name of the email field<br>- `minSubmissionTimeSeconds`: minimum seconds to fill the form (default: 2)<br>- `disposableDomains`: array of blocked email domains |
| Sticky Note3 | Sticky Note | Documents detection logic behavior |  |  | ### Step 3: Detect Spam<br>The Code node runs three checks:<br>1. **Honeypot**: Is the hidden field filled? → Bot<br>2. **Timing**: Was the form submitted in < 2 seconds? → Bot<br>3. **Disposable email**: Is the email domain on the blocklist? → Spam<br><br>Output: `{ isSpam, reasons[], formData }`, reasons array explains why it was flagged. |
| Sticky Note4 | Sticky Note | Documents branch outcomes and extension point |  |  | ### Step 4: Route Result<br>**Spam branch** (top): Returns `200 OK` with a generic thank-you message. The bot thinks the form worked, but nothing is forwarded. This prevents bots from retrying.<br><br>**Legit branch** (bottom): Returns `200 OK` with success status and the cleaned form data.<br><br>**Add your own nodes** after the IF on the legit branch to forward submissions to email, Slack, a CRM, or a database. |
| Receive Form Submission | Webhook | Receives POSTed form submissions and waits for a response node |  | Configure Spam Rules | ### Step 1: Receive Form Data<br>POST request with JSON body:<br><pre><code>{<br>  "name": "Max Mustermann",<br>  "email": "max@example.com",<br>  "message": "Hello!",<br>  "website_url": "",<br>  "_timestamp": "2026-03-13T10:00:00Z"<br>}</code></pre><br>**Frontend HTML example:**<br><pre><code>&lt;!-- Honeypot (hidden from users, bots fill it) --&gt;<br>&lt;div style="position:absolute;left:-9999px;"<br>     aria-hidden="true"&gt;<br>  &lt;input type="text" name="website_url"<br>         tabindex="-1" autocomplete="off"&gt;<br>&lt;/div&gt;<br><br>&lt;!-- Timestamp (set on page load) --&gt;<br>&lt;input type="hidden" name="_timestamp" id="ts"&gt;<br>&lt;script&gt;<br>  document.getElementById('ts').value<br>    = new Date().toISOString();<br>&lt;/script&gt;</code></pre> |
| Configure Spam Rules | Set | Defines anti-spam configuration values | Receive Form Submission | Detect Spam | ### Step 2: Configure Spam Rules<br>Edit the **Configure Spam Rules** node to match your form:<br>- `honeypotFieldName`: name of the hidden honeypot field<br>- `timestampFieldName`: name of the hidden timestamp field<br>- `emailFieldName`: name of the email field<br>- `minSubmissionTimeSeconds`: minimum seconds to fill the form (default: 2)<br>- `disposableDomains`: array of blocked email domains |
| Detect Spam | Code | Evaluates honeypot, timing, and disposable email checks | Configure Spam Rules | Is Spam? | ### Step 3: Detect Spam<br>The Code node runs three checks:<br>1. **Honeypot**: Is the hidden field filled? → Bot<br>2. **Timing**: Was the form submitted in < 2 seconds? → Bot<br>3. **Disposable email**: Is the email domain on the blocklist? → Spam<br><br>Output: `{ isSpam, reasons[], formData }`, reasons array explains why it was flagged. |
| Is Spam? | IF | Routes submissions based on `isSpam` | Detect Spam | Silent OK (Spam Blocked), Forward & Respond (Legit) | ### Step 4: Route Result<br>**Spam branch** (top): Returns `200 OK` with a generic thank-you message. The bot thinks the form worked, but nothing is forwarded. This prevents bots from retrying.<br><br>**Legit branch** (bottom): Returns `200 OK` with success status and the cleaned form data.<br><br>**Add your own nodes** after the IF on the legit branch to forward submissions to email, Slack, a CRM, or a database. |
| Silent OK (Spam Blocked) | Respond to Webhook | Returns generic success response for spam submissions | Is Spam? |  | ### Step 4: Route Result<br>**Spam branch** (top): Returns `200 OK` with a generic thank-you message. The bot thinks the form worked, but nothing is forwarded. This prevents bots from retrying.<br><br>**Legit branch** (bottom): Returns `200 OK` with success status and the cleaned form data.<br><br>**Add your own nodes** after the IF on the legit branch to forward submissions to email, Slack, a CRM, or a database. |
| Forward & Respond (Legit) | Respond to Webhook | Returns success response with cleaned form data for legitimate submissions | Is Spam? |  | ### Step 4: Route Result<br>**Spam branch** (top): Returns `200 OK` with a generic thank-you message. The bot thinks the form worked, but nothing is forwarded. This prevents bots from retrying.<br><br>**Legit branch** (bottom): Returns `200 OK` with success status and the cleaned form data.<br><br>**Add your own nodes** after the IF on the legit branch to forward submissions to email, Slack, a CRM, or a database. |
| Sticky Note5 | Sticky Note | Documents honeypot frontend implementation rationale |  |  | ## **Why this works:**<br>- `position:absolute; left:-9999px` hides the field visually but bots still see it in the DOM<br>- `aria-hidden` keeps screen readers from reading it<br>- `tabindex="-1"` prevents keyboard navigation into it<br>- Do NOT use `display:none`, some bots detect and skip those<br>- Field names are configurable in Step 2 |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Open n8n and create a blank workflow.
   - Name it something like: `Filter spam from webhook form submissions with honeypot and bot detection`.

2. **Add the Webhook trigger**
   - Create a **Webhook** node.
   - Rename it to **Receive Form Submission**.
   - Configure:
     - **HTTP Method:** `POST`
     - **Path:** `form-submit`
     - **Response Mode:** `Using Respond to Webhook Node` / `responseNode`
   - Leave additional options empty unless you need custom authentication or CORS behavior.
   - This node will become the public endpoint your frontend posts to.

3. **Prepare the expected frontend payload**
   - Ensure your website form sends JSON like:
     - `name`
     - `email`
     - `message`
     - `website_url` as hidden honeypot
     - `_timestamp` as hidden timestamp
   - Example body:
     ```json
     {
       "name": "Max Mustermann",
       "email": "max@example.com",
       "message": "Hello!",
       "website_url": "",
       "_timestamp": "2026-03-13T10:00:00Z"
     }
     ```
   - Frontend implementation notes:
     - Put the honeypot input off-screen, not `display:none`
     - Set `_timestamp` on page load with JavaScript
     - Send the form via POST to the webhook URL

4. **Add the spam configuration node**
   - Create a **Set** node.
   - Rename it to **Configure Spam Rules**.
   - Connect **Receive Form Submission → Configure Spam Rules**.
   - Configure the Set node to output **raw JSON**.
   - Paste or recreate this configuration structure:
     - `honeypotFieldName`: `website_url`
     - `timestampFieldName`: `_timestamp`
     - `emailFieldName`: `email`
     - `minSubmissionTimeSeconds`: `2`
     - `disposableDomains`: array of blocked domains
   - Use this domain list:
     - `mailinator.com`
     - `guerrillamail.com`
     - `tempmail.com`
     - `throwaway.email`
     - `yopmail.com`
     - `sharklasers.com`
     - `guerrillamailblock.com`
     - `grr.la`
     - `discard.email`
     - `trashmail.com`
     - `10minutemail.com`
     - `temp-mail.org`
     - `fakeinbox.com`
     - `mailnesia.com`
     - `maildrop.cc`
     - `dispostable.com`
     - `getairmail.com`
     - `mohmal.com`
     - `crazymailing.com`
   - Keep domain entries lowercase.

5. **Add the Code node for detection**
   - Create a **Code** node.
   - Rename it to **Detect Spam**.
   - Connect **Configure Spam Rules → Detect Spam**.
   - Set the node to JavaScript mode.
   - Insert logic equivalent to:
     - read form body from the webhook node
     - read spam config from the Set node
     - initialize `isSpam = false` and `reasons = []`
     - if honeypot field is non-empty, mark spam
     - if timestamp exists and elapsed time is less than threshold, mark spam
     - if email domain is in blocklist, mark spam
     - build `formData` without honeypot and timestamp fields
     - return one JSON item with `isSpam`, `reasons`, `formData`

   - Use these node references in expressions inside the code:
     - webhook data: `$('Receive Form Submission').first().json.body`
     - config data: `$('Configure Spam Rules').first().json`

   - Recreate this behavior precisely:
     - if no valid input object exists, return:
       - `isSpam: false`
       - `reasons: ['No form data received']`
       - `formData: {}`
     - timing check only flags when elapsed seconds are `>= 0` and `< minSubmissionTimeSeconds`
     - email-domain check uses exact match against the configured array
     - `formData` removes only the honeypot and timestamp fields

6. **Add the IF node**
   - Create an **IF** node.
   - Rename it to **Is Spam?**
   - Connect **Detect Spam → Is Spam?**
   - Configure one condition:
     - Left value: `{{ $json.isSpam }}`
     - Operator: `equals`
     - Right value: `true`
     - Type: boolean
   - Keep strict type validation enabled if available.

7. **Add the spam response node**
   - Create a **Respond to Webhook** node.
   - Rename it to **Silent OK (Spam Blocked)**.
   - Connect it to the **true** output of **Is Spam?**
   - Configure:
     - **Respond With:** JSON
     - **Response Headers:** `Content-Type = application/json`
     - **Response Body:**
       ```json
       {
         "success": true,
         "message": "Thank you for your submission."
       }
       ```
   - This intentionally does not reveal that the submission was blocked.

8. **Add the legit response node**
   - Create another **Respond to Webhook** node.
   - Rename it to **Forward & Respond (Legit)**.
   - Connect it to the **false** output of **Is Spam?**
   - Configure:
     - **Respond With:** JSON
     - **Response Headers:** `Content-Type = application/json`
     - **Response Body:** an expression that returns:
       - `success: true`
       - `message: 'Submission received.'`
       - `data: $json.formData`
   - Equivalent expression:
     ```javascript
     {{ JSON.stringify({ success: true, message: 'Submission received.', data: $json.formData }) }}
     ```

9. **Optionally extend the legit branch**
   - If you want real submissions forwarded elsewhere, add nodes between **Is Spam?** false output and **Forward & Respond (Legit)**, or branch off from the legit path.
   - Typical additions:
     - Email node
     - Slack node
     - CRM node
     - Database node
   - Ensure the workflow still always reaches a **Respond to Webhook** node.

10. **Add documentation sticky notes**
   - Create sticky notes for maintainability if desired.
   - Suggested notes:
     - overall workflow explanation
     - example request payload
     - spam rule meanings
     - why the honeypot HTML implementation works
     - branch explanation for spam vs legit responses

11. **Activate the workflow**
   - Save the workflow.
   - Activate it so the production webhook endpoint becomes available.

12. **Test a legitimate submission**
   - Send a POST request with:
     - empty honeypot field
     - valid email not in blocklist
     - timestamp older than 2 seconds
   - Expected result:
     - response from **Forward & Respond (Legit)**
     - includes cleaned `formData`
     - excludes `website_url` and `_timestamp`

13. **Test spam scenarios**
   - **Honeypot test:** send a non-empty `website_url`
   - **Timing test:** send `_timestamp` equal to current time so submission appears under 2 seconds
   - **Disposable email test:** use an email from the blocklist
   - Expected result in all cases:
     - response from **Silent OK (Spam Blocked)**
     - generic success message
     - nothing forwarded downstream

14. **Validate operational constraints**
   - Make sure the frontend sends JSON in the structure expected by the webhook.
   - Confirm field names exactly match the Set node configuration.
   - Keep in mind there are **no external credentials required** for the workflow as provided.
   - There are **no sub-workflows** in this design.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: Florian Eiche | [https://eiche-digital.de](https://eiche-digital.de) |
| Honeypot implementation should be visually hidden off-screen rather than using `display:none` | Frontend form implementation guidance |
| The workflow is intentionally silent when spam is detected to discourage bot retries | Response strategy |
| The natural integration point for email, Slack, CRM, or database delivery is after the legit branch of the IF node | Extension guidance |