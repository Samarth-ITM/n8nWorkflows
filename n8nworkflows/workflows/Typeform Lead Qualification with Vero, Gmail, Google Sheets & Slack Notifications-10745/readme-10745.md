Typeform Lead Qualification with Vero, Gmail, Google Sheets & Slack Notifications

https://n8nworkflows.xyz/workflows/typeform-lead-qualification-with-vero--gmail--google-sheets---slack-notifications-10745


# Typeform Lead Qualification with Vero, Gmail, Google Sheets & Slack Notifications

---

### 1. Workflow Overview

This workflow automates lead qualification and onboarding by integrating Typeform, Vero, Gmail, Google Sheets, and Slack. It listens for new Typeform submissions containing lead information, validates the lead's email, normalizes and enriches data fields, then upserts the lead into Vero CRM. Leads surpassing a configurable qualification score automatically receive a personalized welcome email, get logged into Google Sheets for reporting, and trigger a Slack notification for the sales or marketing team.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Captures new lead submissions from Typeform.
- **1.2 Configuration Setup**: Defines key workflow parameters (e.g., qualification score threshold, Slack channel, email subject).
- **1.3 Email Validation**: Checks the submitted email format using regex to flag invalid emails early.
- **1.4 Data Mapping**: Normalizes and prepares lead data fields for downstream CRM and notification processes.
- **1.5 CRM Synchronization**: Upserts or updates the lead profile in Vero with enriched attributes.
- **1.6 Lead Qualification Check**: Evaluates if the lead meets the qualification score threshold.
- **1.7 Qualified Lead Actions**: For qualified leads, sends a personalized welcome email, logs activity to Google Sheets, and notifies a Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for new Typeform form submissions and outputs the raw response data for processing.

- **Nodes Involved:**  
  - *Typeform Trigger*

- **Node Details:**  
  - **Type:** Trigger node (Typeform Trigger)  
  - **Configuration:** Listens to form ID `rcM5Z5Le`, capturing all answers without simplification.  
  - **Key Variables:** Outputs the full form response JSON, including answers for email, first name, company, score, and consent.  
  - **Input/Output:** No input (trigger); outputs raw Typeform submission JSON.  
  - **Edge Cases:** Potential webhook connectivity issues or delays; form ID misconfiguration; missing fields in response.  
  - **Sticky Note:** Describes role as listener for new Typeform responses.

#### 1.2 Configuration Setup

- **Overview:**  
  Centralizes key workflow variables—qualification score threshold, welcome email subject, and Slack channel—to allow easy adjustments.

- **Nodes Involved:**  
  - *Workflow Configuration*

- **Node Details:**  
  - **Type:** Set node  
  - **Configuration:**  
    - `qualificationScore` = 40 (minimum score to qualify a lead)  
    - `welcomeEmailSubject` = "Welcome to Our Platform!"  
    - `slackChannel` = "#leads"  
  - **Key Expressions:** Static values assigned; accessible downstream for decision and messaging nodes.  
  - **Input/Output:** Input from Typeform Trigger; outputs enriched JSON with config variables.  
  - **Edge Cases:** User must update these values to reflect desired thresholds; otherwise, default values apply.

#### 1.3 Email Validation

- **Overview:**  
  Validates the submitted email address field using a regex pattern to detect format correctness.

- **Nodes Involved:**  
  - *Validate Email Format*

- **Node Details:**  
  - **Type:** Code node (JavaScript)  
  - **Configuration:** Runs once per item; checks if `email` field matches regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`  
  - **Key Expressions:**  
    - Reads email from `$input.item.json.email`  
    - Adds `isValidEmail` boolean and `validationStatus` string (`'valid'` or `'invalid'`) to output JSON  
  - **Input/Output:** Input: JSON with email; Output: JSON extended with validation flags.  
  - **Edge Cases:** Missing or empty email field results in invalid flag; regex might not cover all valid email edge cases but sufficient for general validation.

#### 1.4 Data Mapping

- **Overview:**  
  Normalizes raw Typeform fields into a consistent contact schema for CRM and logging.

- **Nodes Involved:**  
  - *Map Contact Fields*

- **Node Details:**  
  - **Type:** Set node  
  - **Configuration:** Assigns fields with fallback logic:  
    - `email`: empty string (to be assigned later)  
    - `firstName`: from `first_name` or `firstName` or empty  
    - `company`: from `company` or empty  
    - `plan`: from `plan` or default `"Free"`  
    - `score`: from `score` or default `50`  
    - `source`: `"Typeform"` (static)  
    - `consent`: from `consent` or default `true`  
  - **Input/Output:** Takes validated email JSON; outputs normalized contact JSON.  
  - **Edge Cases:** Missing fields replaced with reasonable defaults; ensures downstream nodes receive consistent structure.

#### 1.5 CRM Synchronization

- **Overview:**  
  Upserts or updates the lead profile in Vero using email as the unique identifier and syncing enriched attributes.

- **Nodes Involved:**  
  - *Vero Upsert Profile*

- **Node Details:**  
  - **Type:** Vero node  
  - **Configuration:**  
    - Uses lead’s email as `id`  
    - Sets additional fields: email, firstName, company, plan, score, source, consent  
  - **Key Expressions:** Reads fields mainly from `$json` and `form_response.answers`  
  - **Input/Output:** Input from mapped contact fields; output passes lead data for qualification check.  
  - **Edge Cases:** Vero API errors (authentication, rate limits); missing email would prevent upsert; data type mismatches.  
  - **Sticky Note:** Notes that this node creates or updates Vero customer profiles.

#### 1.6 Lead Qualification Check

- **Overview:**  
  Compares lead’s score against configured qualification threshold to determine if lead is qualified.

- **Nodes Involved:**  
  - *IF Qualified Lead*

- **Node Details:**  
  - **Type:** If node  
  - **Configuration:** Condition checks if `score` from "Map Contact Fields" is greater than or equal to `qualificationScore` from configuration (40 by default).  
  - **Input/Output:** Input from Vero Upsert Profile; routes qualified leads to welcome email path.  
  - **Edge Cases:** Score missing or non-numeric may cause condition failure; leads not meeting threshold are silently discarded or could be routed elsewhere if extended.  
  - **Sticky Note:** Explains routing logic based on qualification score.

#### 1.7 Qualified Lead Actions

- **Overview:**  
  For leads passing qualification, sends a personalized welcome email, logs the event to Google Sheets, and notifies a Slack channel.

- **Nodes Involved:**  
  - *Send Welcome Email*  
  - *Log to Google Sheets*  
  - *Notify Slack Channel*

- **Node Details:**  

  - **Send Welcome Email**  
    - **Type:** Gmail node  
    - **Configuration:**  
      - Sends to email from first Typeform answer  
      - Subject from workflow configuration (`welcomeEmailSubject`)  
      - Message includes first name and plan dynamically injected  
    - **Edge Cases:** Gmail API quota or auth issues; missing email address; message template errors.  
    - **Sticky Note:** Describes sending personalized welcome messages.

  - **Log to Google Sheets**  
    - **Type:** Google Sheets node  
    - **Configuration:**  
      - Appends a row to sheet named `vero` in a specified spreadsheet ID  
      - Columns logged: timestamp, email, firstName, company, plan, score, validation status, action taken (welcome_sent)  
    - **Edge Cases:** API permission errors; sheet name or document ID mistakes; data type conversions.  
    - **Sticky Note:** Notes building an auditable trail for reporting.

  - **Notify Slack Channel**  
    - **Type:** Slack node  
    - **Configuration:**  
      - Posts a formatted message to configured Slack channel (`#general` by default, though configuration node says `#leads`)  
      - Message includes lead details: email, name, company, plan, score, and action taken  
      - Uses OAuth2 authentication  
    - **Edge Cases:** Slack API limits; invalid channel name; OAuth token expiry.  
    - **Sticky Note:** Explains instant team visibility via Slack notification.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                            | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                      |
|---------------------|---------------------|------------------------------------------|-----------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Typeform Trigger    | Typeform Trigger    | Captures new Typeform submissions         | None                  | Workflow Configuration       | Listens for new Typeform responses from the specified form.                                   |
| Workflow Configuration | Set                | Stores key configurable variables          | Typeform Trigger       | Validate Email Format        | Centralizes user-editable variables (qualification score, Slack channel, email subject).       |
| Validate Email Format | Code                | Validates email format via regex           | Workflow Configuration | Map Contact Fields           | Runs regex check on email; adds isValidEmail and validationStatus flags.                       |
| Map Contact Fields  | Set                 | Normalizes and maps contact data           | Validate Email Format  | Vero Upsert Profile          | Normalizes Typeform fields into clean contact payload for CRM and logging.                     |
| Vero Upsert Profile | Vero                | Creates/updates lead profile in Vero CRM   | Map Contact Fields     | IF Qualified Lead            | Creates or updates a Vero customer profile using email as ID; syncs lead attributes.          |
| IF Qualified Lead   | If                  | Checks if lead score meets qualification   | Vero Upsert Profile    | Send Welcome Email           | Routes leads by qualification score to welcome email and notification path.                    |
| Send Welcome Email  | Gmail                | Sends personalized welcome email           | IF Qualified Lead      | Log to Google Sheets         | Sends personalized welcome message to qualified leads.                                        |
| Log to Google Sheets | Google Sheets        | Logs lead event details for reporting       | Send Welcome Email     | Notify Slack Channel         | Appends event to Google Sheets for audit trail and reporting.                                 |
| Notify Slack Channel | Slack                | Sends Slack notification about qualified lead | Log to Google Sheets   | None                        | Posts concise summary of qualified lead to configured Slack channel.                           |
| Sticky Note         | Sticky Note          | Documentation notes                         | None                  | None                        | Various notes explaining node roles (duplicated across respective nodes).                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add "Typeform Trigger" node**:  
   - Type: Typeform Trigger  
   - Parameters:  
     - Form ID: `rcM5Z5Le` (replace with your form ID)  
     - Only Answers: false  
     - Simplify Answers: false  
   - This node will listen for new form submissions.

3. **Add "Workflow Configuration" node** (Set):  
   - Connect from Typeform Trigger  
   - Define variables:  
     - `qualificationScore` = 40 (number)  
     - `welcomeEmailSubject` = "Welcome to Our Platform!" (string)  
     - `slackChannel` = "#leads" (string)  
   - Include other fields to pass through input JSON.

4. **Add "Validate Email Format" node** (Code):  
   - Connect from Workflow Configuration  
   - Run once per item  
   - JavaScript code:  
     ```js
     const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
     const email = $input.item.json.email || '';
     const isValidEmail = emailRegex.test(email);

     return {
       ...($input.item.json),
       isValidEmail,
       validationStatus: isValidEmail ? 'valid' : 'invalid'
     };
     ```
   - This checks email format and adds validation flags.

5. **Add "Map Contact Fields" node** (Set):  
   - Connect from Validate Email Format  
   - Map fields with fallback expressions:  
     - `email`: (empty string)  
     - `firstName`: `{{$json.first_name || $json.firstName || ''}}`  
     - `company`: `{{$json.company || ''}}`  
     - `plan`: `{{$json.plan || 'Free'}}`  
     - `score`: `{{$json.score || 50}}`  
     - `source`: `"Typeform"` (static)  
     - `consent`: `{{$json.consent || true}}`  
   - Include other fields.

6. **Add "Vero Upsert Profile" node**:  
   - Connect from Map Contact Fields  
   - Configure credentials for Vero API.  
   - Parameters:  
     - `id`: `{{$json.email}}` (unique identifier)  
     - Additional Fields:  
       - `email`: `{{$json.form_response.answers[0].email}}`  
     - Data Attributes:  
       - `firstName`: `{{$json.form_response.answers[1].text}}`  
       - `company`: `{{$json.form_response.answers[2].text}}`  
       - `plan`: `{{$json.plan}}`  
       - `score`: `{{$json.score}}`  
       - `source`: `{{$json.source}}`  
       - `consent`: `{{$json.consent}}`

7. **Add "IF Qualified Lead" node** (If):  
   - Connect from Vero Upsert Profile  
   - Condition:  
     - Left value: `{{$node["Map Contact Fields"].json.score}}`  
     - Operation: `>=`  
     - Right value: `{{$node["Workflow Configuration"].json.qualificationScore}}`  
   - This filters leads by qualification score.

8. **Add "Send Welcome Email" node** (Gmail):  
   - Connect from IF Qualified Lead (True branch)  
   - Configure Gmail OAuth2 credentials.  
   - Parameters:  
     - Send To: `={{ $node["Typeform Trigger"].json.form_response.answers[0].email }}`  
     - Subject: `={{ $node["Workflow Configuration"].json.welcomeEmailSubject }}`  
     - Message (HTML):  
       ```
       Hi {{ $node["Typeform Trigger"].json.form_response.answers[1].text }},<br><br>
       Welcome to our platform! We're excited to have you on board.<br><br>
       Your plan: {{ $node["Map Contact Fields"].json.plan }} <br><br>
       Best regards,<br>The Team
       ```

9. **Add "Log to Google Sheets" node**:  
   - Connect from Send Welcome Email  
   - Configure Google Sheets credentials with access to target spreadsheet.  
   - Parameters:  
     - Operation: Append  
     - Sheet Name: `"vero"`  
     - Document ID: (your Google Sheet document ID)  
     - Columns/Values to map:  
       - `timestamp`: Current ISO timestamp (`{{$now.toISO()}}`)  
       - `email`, `firstName`, `company`, `plan`, `score`, `status` (validationStatus), `action` ("welcome_sent")  
     - Mapping mode: Define below.

10. **Add "Notify Slack Channel" node**:  
    - Connect from Log to Google Sheets  
    - Configure Slack OAuth2 credentials.  
    - Parameters:  
      - Channel: `#general` (or use `{{$node["Workflow Configuration"].json.slackChannel}}` for dynamic)  
      - Text message:  
        ```
        🎉 New qualified lead!

        *Email:* {{ $json.email }}
        *Name:* {{ $json.firstName }}
        *Company:* {{ $json.company }}
        *Plan:* {{ $json.plan }}
        *Score:* {{ $json.score }}
        *Action:* Welcome email sent
        ```

11. **Test the workflow** by submitting a test entry to the Typeform; observe full flow including email validation, CRM upsert, email sending, logging, and Slack notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                              |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| This workflow captures Typeform leads, validates emails, upserts to Vero, sends welcome emails, logs to Sheets, and alerts Slack. | Project purpose overview (from Sticky Note9).                                               |
| Adjust qualification score, Slack channel, and email subject centrally in Workflow Configuration node for easy customization. | Workflow Configuration node description (Sticky Note1).                                    |
| Regex email validation is a simple format check, not exhaustive email verification.                            | Email validation logic explanation (Sticky Note2).                                         |
| Google Sheets provides an auditable log for lead activities, supporting reporting and stakeholder review.     | Google Sheets logging rationale (Sticky Note7).                                            |
| Slack notifications give instant team visibility without manual CRM checks.                                   | Slack notification note (Sticky Note8).                                                    |
| Vero integration ensures leads are segmented and enriched for marketing automation downstream.                 | Vero CRM syncing explanation (Sticky Note4).                                               |

---

**Disclaimer:** The text provided is derived solely from an n8n workflow automation. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.