Verify form leads in real time with NeverBounce before adding to your CRM

https://n8nworkflows.xyz/workflows/verify-form-leads-in-real-time-with-neverbounce-before-adding-to-your-crm-14060


# Verify form leads in real time with NeverBounce before adding to your CRM

# 1. Workflow Overview

This workflow implements a real-time email validation gate in front of CRM lead creation. Its purpose is to prevent invalid email addresses from entering downstream systems by checking each submitted email with NeverBounce before the lead is accepted.

Typical use cases:
- Website lead capture forms
- Demo request forms
- Newsletter or contact forms that feed a CRM
- Any scenario where email quality must be enforced before record creation

The workflow has a simple but effective branching structure:
- A user submits a form
- The email is validated through NeverBounce
- If the email is valid, the workflow continues to CRM creation
- If the email is invalid, the user is shown a retry form instead of being accepted

## 1.1 Input Reception

The workflow starts with an n8n form trigger that collects an email address from the user.

## 1.2 Email Verification

The submitted email is sent to the NeverBounce email verification node, which returns a verification result including a status value.

## 1.3 Decision and Routing

An If node checks whether the NeverBounce status is exactly `valid`. Based on that result, the workflow branches into a success path or a retry path.

## 1.4 Lead Creation and Confirmation

For valid emails, the workflow proceeds to a placeholder CRM step and then displays a success completion screen.

## 1.5 Error and Retry Loop

For invalid emails, the workflow displays a form asking the user to correct the email, then resubmits the updated value back into the verification step.

---

# 2. Block-by-Block Analysis

## Block 1 — Form Intake

### Overview
This block receives the initial user input through an n8n-hosted form. It is the main entry point of the workflow and provides the email value used throughout the rest of the process.

### Nodes Involved
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes an n8n form/webhook and starts the workflow when a user submits the form.
- **Configuration choices:**
  - Form title is set to **Verify**
  - One form field is defined:
    - Label: **Email**
- **Key expressions or variables used:**
  - Downstream nodes reference the submitted field as `{{$json.Email}}`
- **Input and output connections:**
  - No input; this is a trigger node
  - Output goes to **Neverbounce: verify an email address**
- **Version-specific requirements:**
  - Uses **typeVersion 2.5**
  - Requires a recent n8n version that supports Form Trigger v2.x behavior
- **Edge cases or potential failure types:**
  - If the form field name changes, downstream references to `$json.Email` may break
  - If the production/test form URL is misused, submissions may not reach the intended environment
  - If n8n instance access or webhook exposure is misconfigured, the form will not be reachable
- **Sub-workflow reference:**
  - None

---

## Block 2 — Email Verification

### Overview
This block sends the submitted email to NeverBounce for validation. It acts as the quality gate for the workflow and determines whether the lead can proceed.

### Nodes Involved
- Neverbounce: verify an email address

### Node Details

#### Neverbounce: verify an email address
- **Type and technical role:** `n8n-nodes-neverbounce-email-verification.nbEmailVerification`  
  Third-party integration node that validates an email address through the NeverBounce API.
- **Configuration choices:**
  - Email input field is set dynamically from the form submission
  - No additional fields are configured
- **Key expressions or variables used:**
  - `={{ $json.Email }}`
  - Downstream logic expects a response structure containing `verification_result.status`
- **Input and output connections:**
  - Input from:
    - **On form submission**
    - **Form: Error & Retry**
  - Output to:
    - **Check if Status = Valid**
- **Version-specific requirements:**
  - Uses **typeVersion 1**
  - Requires the NeverBounce community/integration node to be installed and available in the n8n instance
- **Edge cases or potential failure types:**
  - Missing or invalid NeverBounce credentials
  - API quota limits or billing restrictions
  - Network/API timeout
  - Unexpected API response shape causing downstream expression failures
  - Empty or malformed email input from the form
- **Sub-workflow reference:**
  - None

---

## Block 3 — Decision Logic

### Overview
This block determines whether the verification result should be accepted. It is currently configured to allow only emails whose status is exactly `valid`.

### Nodes Involved
- Check if Status = Valid

### Node Details

#### Check if Status = Valid
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router that splits the workflow into valid and invalid branches.
- **Configuration choices:**
  - One string equality condition is configured
  - Left value: NeverBounce status
  - Right value: `valid`
- **Key expressions or variables used:**
  - `={{ $json.verification_result.status }}`
- **Input and output connections:**
  - Input from:
    - **Neverbounce: verify an email address**
  - True output goes to:
    - **Create Lead in CRM**
  - False output goes to:
    - **Form: Error & Retry**
- **Version-specific requirements:**
  - Uses **typeVersion 2.3**
- **Edge cases or potential failure types:**
  - If `verification_result.status` is missing, the comparison may evaluate unexpectedly
  - Other statuses such as `catchall`, `invalid`, `disposable`, or `unknown` are all treated as failure in the current configuration
  - Future API response changes could break the condition
- **Sub-workflow reference:**
  - None

---

## Block 4 — Success Path

### Overview
This block handles accepted leads. It currently contains a placeholder CRM node followed by a completion screen shown to the user.

### Nodes Involved
- Create Lead in CRM
- Form: Success Message

### Node Details

#### Create Lead in CRM
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Placeholder node used to mark where CRM insertion should happen.
- **Configuration choices:**
  - No parameters; intentionally empty
- **Key expressions or variables used:**
  - None currently, but in a real CRM node you would typically map the email field and any other submitted form data
- **Input and output connections:**
  - Input from:
    - **Check if Status = Valid** (true branch)
  - Output to:
    - **Form: Success Message**
- **Version-specific requirements:**
  - Uses **typeVersion 1**
- **Edge cases or potential failure types:**
  - NoOp itself does not fail meaningfully
  - When replaced with a CRM node, expected issues include auth errors, field mapping problems, duplicate record errors, API throttling, and validation failures
- **Sub-workflow reference:**
  - None

#### Form: Success Message
- **Type and technical role:** `n8n-nodes-base.form`  
  Completion-response node that renders a success message after a valid submission has been processed.
- **Configuration choices:**
  - Operation is set to **completion**
  - Title: **Form submitted successfully**
  - Message: **Thank you for your submission**
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input from:
    - **Create Lead in CRM**
  - No downstream output in this workflow
- **Version-specific requirements:**
  - Uses **typeVersion 2.5**
- **Edge cases or potential failure types:**
  - If used incorrectly outside a form-driven execution context, the expected user-facing response behavior may not occur
  - If a real CRM node fails before this step, the user may never see the success confirmation
- **Sub-workflow reference:**
  - None

---

## Block 5 — Error Handling and Retry

### Overview
This block handles invalid emails by showing the user a retry form. The corrected input loops back into the verification node, allowing repeated attempts until a valid email is provided.

### Nodes Involved
- Form: Error & Retry

### Node Details

#### Form: Error & Retry
- **Type and technical role:** `n8n-nodes-base.form`  
  Interactive form node that returns a new form screen prompting the user to re-enter the email.
- **Configuration choices:**
  - One form field is defined
  - Field name: **Email**
  - Field label: **⚠️ Invalid email. Please check for typos and try again.**
- **Key expressions or variables used:**
  - The field name remains `Email`, which preserves compatibility with `{{$json.Email}}` in the verification node
- **Input and output connections:**
  - Input from:
    - **Check if Status = Valid** (false branch)
  - Output to:
    - **Neverbounce: verify an email address**
- **Version-specific requirements:**
  - Uses **typeVersion 2.5**
- **Edge cases or potential failure types:**
  - If the field name were changed from `Email`, the retry loop would break unless the NeverBounce expression is updated
  - Users may remain stuck in a loop if they repeatedly enter invalid addresses
  - No max retry count or alternate escalation path is implemented
- **Sub-workflow reference:**
  - None

---

## Additional Node — Visual Documentation

### Overview
This node does not affect execution. It provides setup guidance, implementation notes, and a video resource directly inside the workflow canvas.

### Nodes Involved
- Sticky Note

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation-only canvas annotation for human operators.
- **Configuration choices:**
  - Large note placed around the workflow
  - Contains setup instructions, behavior notes, and a YouTube reference
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - Uses **typeVersion 1**
- **Edge cases or potential failure types:**
  - No runtime effect
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| On form submission | Form Trigger | Receives the initial email submission and starts the workflow |  | Neverbounce: verify an email address | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| Neverbounce: verify an email address | NeverBounce Email Verification | Verifies the submitted or retried email via NeverBounce | On form submission; Form: Error & Retry | Check if Status = Valid | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| Check if Status = Valid | If | Routes execution based on NeverBounce validation status | Neverbounce: verify an email address | Create Lead in CRM; Form: Error & Retry | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| Form: Error & Retry | Form | Shows an invalid-email retry form and loops back to verification | Check if Status = Valid | Neverbounce: verify an email address | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| Create Lead in CRM | No Operation | Placeholder for CRM record creation after successful validation | Check if Status = Valid | Form: Success Message | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| Form: Success Message | Form | Displays a final success/completion screen | Create Lead in CRM |  | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Setup Steps: 1. Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Setup Steps: 2. Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Setup Steps: 3. Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Setup Steps: 4. Error Handling: If an email is invalid, the 'Form - Invalid' node triggers a retry screen for the user. |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Setup Steps: 5. Success State: The 'Form - Success' node displays your "Thank You" message after a valid submission. |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Setup Steps: 6. Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). |
| Sticky Note | Sticky Note | In-canvas documentation and setup guidance |  |  | Video: Watch the 75-second setup guide above — https://www.youtube.com/watch?v=FDy-eaITcAQ |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like **NeverBounce Lead Gatekeeper**.

2. **Add the entry form trigger**
   - Add node: **Form Trigger**
   - Rename it to **On form submission**
   - Set:
     - **Form Title**: `Verify`
   - Add one form field:
     - Label: `Email`
   - Keep the output field name aligned with the visible field so downstream data is available as `Email`.

3. **Prepare the form URL**
   - Open the Form Trigger node
   - Copy the **Test URL** if you are testing manually
   - Later, when deploying, use the production URL appropriate for your environment
   - Make sure your website or form source points to the correct URL

4. **Add the NeverBounce verification node**
   - Add node: **NeverBounce Email Verification**
   - Rename it to **Neverbounce: verify an email address**
   - Configure the email field with this expression:
     - `{{$json.Email}}`
   - Leave additional fields empty unless you have a specific need
   - Connect:
     - **On form submission** → **Neverbounce: verify an email address**

5. **Configure NeverBounce credentials**
   - In the NeverBounce node, create or select a credential
   - Provide your **NeverBounce API key**
   - Test the credential if your n8n version supports credential testing
   - Confirm the node can execute successfully with a sample email

6. **Add the conditional decision node**
   - Add node: **If**
   - Rename it to **Check if Status = Valid**
   - Add one condition:
     - Left value: `{{$json.verification_result.status}}`
     - Operator: **equals**
     - Right value: `valid`
   - Connect:
     - **Neverbounce: verify an email address** → **Check if Status = Valid**

7. **Add the CRM placeholder**
   - Add node: **No Operation**
   - Rename it to **Create Lead in CRM**
   - Connect the **true** output of the If node to this node:
     - **Check if Status = Valid** (true) → **Create Lead in CRM**

8. **Replace the CRM placeholder if needed**
   - In a production implementation, replace **Create Lead in CRM** with the appropriate CRM node, for example:
     - HubSpot
     - Salesforce
     - Pipedrive
     - Airtable
     - HTTP Request to a custom API
   - Map at minimum the email field from the form submission
   - If your CRM requires more fields than email, add them to the trigger form first and then map them here

9. **Add the success response node**
   - Add node: **Form**
   - Rename it to **Form: Success Message**
   - Set the operation to **Completion**
   - Configure:
     - **Completion Title**: `Form submitted successfully`
     - **Completion Message**: `Thank you for your submission`
   - Connect:
     - **Create Lead in CRM** → **Form: Success Message**

10. **Add the invalid-email retry form**
    - Add node: **Form**
    - Rename it to **Form: Error & Retry**
    - Add one field:
      - Field Name: `Email`
      - Field Label: `⚠️ Invalid email. Please check for typos and try again.`
    - Important:
      - Keep the field name exactly `Email`
      - This ensures the retry response remains compatible with `{{$json.Email}}` in the NeverBounce node
    - Connect the **false** output of the If node:
      - **Check if Status = Valid** (false) → **Form: Error & Retry**

11. **Create the retry loop**
    - Connect:
      - **Form: Error & Retry** → **Neverbounce: verify an email address**
    - This allows the user to resubmit a corrected email address without leaving the flow

12. **Optionally add a sticky note**
    - Add node: **Sticky Note**
    - Place it around the workflow
    - Include implementation notes such as:
      - Where to paste the form URL
      - Which credential to configure
      - That the If node currently only accepts `valid`
      - That you may expand acceptance to `catchall` if desired
      - That the CRM placeholder should be replaced
      - A team video/reference link if useful

13. **Test the valid path**
    - Submit a known valid email through the form
    - Confirm:
      - The NeverBounce node returns `verification_result.status = valid`
      - The If node follows the true branch
      - The CRM placeholder or real CRM node executes
      - The success completion form appears

14. **Test the invalid path**
    - Submit an obviously invalid email
    - Confirm:
      - The If node follows the false branch
      - The retry form appears
      - Entering a corrected email loops back to NeverBounce
      - A valid correction then continues to the success path

15. **Consider logic adjustments**
    - If you want a less strict workflow, modify the If node logic to also accept statuses such as `catchall`
    - This can be done with:
      - Multiple If conditions
      - A Switch node
      - A Code node for custom policy logic

16. **Publish and activate**
    - Switch from testing URLs to production endpoints
    - Activate the workflow
    - Update your website or application to use the active form/webhook endpoint

## Sub-workflow setup
- This workflow does **not** invoke or depend on any sub-workflow.
- There are no Execute Workflow nodes and no nested workflow parameters to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Goal: Stop lead pollution at the source. This workflow creates a “Gatekeeper” for your web forms, ensuring that only 100% valid email addresses ever reach your CRM. | Workflow purpose |
| Webhook Link: Open 'On form submission' and copy the Test URL to your form settings. | Initial setup guidance |
| Credentials: In 'Neverbounce: verify an email', select your NeverBounce API key. | Credential setup |
| Logic: The 'If' node is set to valid. Add catchall if you want to be less restrictive. | Decision policy |
| Error Handling: If an email is invalid, the retry form node triggers a retry screen for the user. | User feedback flow |
| Success State: The success form node displays the final thank-you message after a valid submission. | Completion flow |
| Final Step: Replace 'Create Lead in CRM' with your actual CRM node (HubSpot, Salesforce, etc.). | Production implementation |
| Watch the 75-second setup guide above | https://www.youtube.com/watch?v=FDy-eaITcAQ |

## Additional implementation note
The retry behavior depends on field-name consistency. Both the initial form and the retry form must provide the email value under the same key, which in this workflow is `Email`. If you rename that field in either place, you must also update the NeverBounce node expression accordingly.