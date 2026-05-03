Qualify inbound leads with Vapi voice AI and log results to Google Sheets

https://n8nworkflows.xyz/workflows/qualify-inbound-leads-with-vapi-voice-ai-and-log-results-to-google-sheets-13948


# Qualify inbound leads with Vapi voice AI and log results to Google Sheets

# 1. Workflow Overview

This workflow captures inbound leads from an n8n form, validates and standardizes the submitted phone number, places an automated qualification call through Vapi Voice AI, waits until the call is finished, and writes the outcome into Google Sheets.

It is designed for teams that want to qualify leads immediately after form submission instead of relying on manual follow-up. The workflow supports three end states:

- **Invalid phone number** → logged as an invalid lead
- **Voicemail reached** → logged for callback
- **Completed AI qualification call** → logged with structured qualification data

## 1.1 Input Reception and Data Standardization
The workflow starts with a public form. Submitted fields are normalized, especially the phone number, so downstream API calls receive a predictable US 10-digit number.

## 1.2 Phone Validation and Early Exit
A decision node checks whether the phone number was standardized successfully. If not, the lead is appended to Google Sheets with an “Incorrect Phone #” status and the qualification call is skipped.

## 1.3 AI Voice Call Initiation
For valid numbers, the workflow sends a POST request to Vapi to start an outbound call. The request includes the assistant ID, phone number ID, customer phone number, and lead-specific variables used by the assistant.

## 1.4 Call Monitoring and Polling
After an initial wait, the workflow polls the Vapi call endpoint repeatedly until the call status becomes `ended`. This loop prevents premature logging when the call is still active.

## 1.5 Outcome Classification and Logging
Once the call is finished, the workflow checks whether the call ended in voicemail. Voicemail calls are logged with fallback values. Otherwise, the workflow extracts structured outputs from the Vapi response and writes a completed qualification row into Google Sheets.

---

# 2. Block-by-Block Analysis

## Block 1 — Form Intake and Phone Standardization

### Overview
This block collects lead information from a hosted n8n form and transforms the submitted phone number into a standardized US 10-digit format. It ensures the next steps can reliably determine whether the lead is callable.

### Nodes Involved
- On form submission
- Standardize Data

### Node Details

#### 1. On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry point that exposes an n8n-hosted form and starts the workflow whenever a visitor submits it.
- **Configuration choices:**
  - Form title: **Work with us!**
  - Form description: **Fill out the form and we will reach out ASAP.**
  - Required fields:
    - Name
    - Phone Number
    - Email
    - Company Name
    - Role
    - Request
    - Company Size
  - `Phone Number` is configured as a **number** field.
  - `Email` is configured as an **email** field.
  - `Company Size` is a dropdown with:
    - 1
    - 2-10
    - 11-50
    - 51-100
    - 101+
- **Key expressions or variables used:** None in the node itself; it emits the submitted form values as JSON.
- **Input and output connections:**
  - Input: none
  - Output: Standardize Data
- **Version-specific requirements:** Uses `typeVersion: 2.4`; ensure your n8n version supports the Form Trigger node with hosted form fields.
- **Edge cases or potential failure types:**
  - Form accessibility/public URL issues
  - Browser-side validation errors for email/required fields
  - Phone field as numeric input may strip formatting and can behave poorly with leading `+` or leading zeroes
- **Sub-workflow reference:** None

#### 2. Standardize Data
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that cleans and validates the phone number.
- **Configuration choices:**
  - Reads all items from input.
  - Converts `Phone Number` to a string.
  - Removes all non-digit characters with regex `\D`.
  - Accepts:
    - exactly 10 digits, or
    - 11 digits starting with `1` and strips the leading `1`
  - Any other pattern becomes `incorrect format`
  - Rewrites the `Phone Number` field in the item JSON.
- **Key expressions or variables used:**
  - `item.json['Phone Number']`
  - `String(phoneNumber).replace(/\D/g, '')`
- **Input and output connections:**
  - Input: On form submission
  - Output: Incorrect Phone?
- **Version-specific requirements:** Uses Code node `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Non-US numbers are always marked invalid
  - Empty or null phone values become invalid
  - Numeric form handling may alter very large numbers or formatting before this node runs
  - If you want international support, the code must be redesigned
- **Sub-workflow reference:** None

---

## Block 2 — Phone Validation and Invalid Lead Logging

### Overview
This block determines whether the phone number passed standardization. Invalid phone numbers are logged to Google Sheets and removed from the calling path.

### Nodes Involved
- Incorrect Phone?
- Log Incorrect Phone

### Node Details

#### 3. Incorrect Phone?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branch that checks whether the standardized phone number is flagged as invalid.
- **Configuration choices:**
  - Condition compares `{{$json["Phone Number"]}}` to the literal string `incorrect format`
  - True branch = invalid number
  - False branch = valid number
- **Key expressions or variables used:**
  - `={{ $json["Phone Number"] }}`
- **Input and output connections:**
  - Input: Standardize Data
  - True output: Log Incorrect Phone
  - False output: Call Lead
- **Version-specific requirements:** Uses If node `typeVersion: 2.3` with conditions version 3.
- **Edge cases or potential failure types:**
  - If the Code node changes its invalid marker text, this condition must match it exactly
  - Strict comparison can fail if the value contains extra whitespace
- **Sub-workflow reference:** None

#### 4. Log Incorrect Phone
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends invalid leads into a Google Sheet for tracking and cleanup.
- **Configuration choices:**
  - Operation: **append**
  - Mapping mode: **define below**
  - Writes fields:
    - Date → `{{$now.format('yyyy-MM-dd hh:mm a')}}`
    - Name → `{{$json.Name}}`
    - Role → `{{$json.Role}}`
    - Email → `{{$json.Email}}`
    - Phone → `{{$json['Phone Number']}}`
    - Status → `Incorrect Phone #`
    - Company → `{{$json['Company Name']}}`
    - Request → `{{$json.Request}}`
    - Company Size → `{{$json['Company Size']}}`
  - `documentId` and `sheetName` are present but blank in the supplied workflow and must be configured
- **Key expressions or variables used:**
  - `={{ $now.format('yyyy-MM-dd hh:mm a') }}`
  - `={{ $json.<field> }}`
- **Input and output connections:**
  - Input: Incorrect Phone? (true branch)
  - Output: none
- **Version-specific requirements:** Uses Google Sheets node `typeVersion: 4.7`.
- **Edge cases or potential failure types:**
  - Missing Google Sheets credentials
  - Invalid spreadsheet URL/document ID
  - Missing tab name
  - Column/header mismatch in the target sheet
  - Date formatting differences depending on n8n runtime support
- **Sub-workflow reference:** None

---

## Block 3 — Vapi Call Initiation

### Overview
This block starts an outbound qualification call to the lead through Vapi. It passes lead metadata to the assistant as variables so the voice agent can personalize the conversation.

### Nodes Involved
- Call Lead

### Node Details

#### 5. Call Lead
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the Vapi API to create a call.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.vapi.ai/call`
  - Authentication: generic credential type using **HTTP Bearer Auth**
  - Body type: JSON
  - JSON body includes:
    - `assistantId`: placeholder `YOUR_VAPI_ASSISTANT_ID`
    - `phoneNumberId`: placeholder `YOUR_VAPI_PHONE_NUMBER_ID`
    - `customers[0].number`: `+1{{ $json['Phone Number'] }}`
    - `assistantOverrides.variableValues`:
      - `lead_name`
      - `lead_company_name`
      - `lead_request`
- **Key expressions or variables used:**
  - `{{ $json['Phone Number'] }}`
  - `{{ $json.Name }}`
  - `{{ $json['Company Name'] }}`
  - `{{ $json.Request }}`
- **Input and output connections:**
  - Input: Incorrect Phone? (false branch)
  - Output: Wait
- **Version-specific requirements:** HTTP Request node `typeVersion: 4.3`.
- **Edge cases or potential failure types:**
  - Missing/invalid Vapi API key in bearer auth
  - Placeholder assistant/phone number IDs not replaced
  - Vapi rejecting malformed phone number
  - API rate limits or transient HTTP 4xx/5xx responses
  - If Vapi returns a structure different from expected, downstream `Get Call Details` may fail
- **Sub-workflow reference:** None

---

## Block 4 — Wait and Poll Until Call Completion

### Overview
This block gives the call time to happen, then repeatedly checks Vapi for the call status until the call is marked as ended. It acts as a polling loop.

### Nodes Involved
- Wait
- Get Call Details
- Limit
- Ended?
- Polling

### Node Details

#### 6. Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before the first call-status check.
- **Configuration choices:**
  - Wait amount: `60`
  - No explicit unit is shown in the JSON snippet; in practice this node is intended as a 60-second delay based on the accompanying sticky note
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: Call Lead
  - Output: Get Call Details
- **Version-specific requirements:** Wait node `typeVersion: 1.1`.
- **Edge cases or potential failure types:**
  - If configured with the wrong unit, polling cadence may be incorrect
  - Long waits can tie up execution state
  - If call durations vary significantly, 60 seconds may be too short or too long
- **Sub-workflow reference:** None

#### 7. Get Call Details
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves the current state of the specific Vapi call.
- **Configuration choices:**
  - URL expression: `=https://api.vapi.ai/call/{{ $json.results[0].id }}`
  - Authentication: generic credential type with HTTP Bearer Auth
  - `executeOnce: true`
- **Key expressions or variables used:**
  - `{{ $json.results[0].id }}`
- **Input and output connections:**
  - Inputs:
    - Wait
    - Polling
  - Output: Limit
- **Version-specific requirements:** HTTP Request node `typeVersion: 4.3`.
- **Edge cases or potential failure types:**
  - Assumes the call creation response contains `results[0].id`
  - If Vapi returns `id` directly instead of nested inside `results`, this breaks
  - On later polling iterations, the input payload shape must still provide the needed call ID or this expression will fail
  - API auth and network errors remain possible
- **Sub-workflow reference:** None

#### 8. Limit
- **Type and technical role:** `n8n-nodes-base.limit`  
  Restricts items passed onward, likely to enforce single-item processing.
- **Configuration choices:**
  - No explicit parameters set, so default item limiting behavior applies
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: Get Call Details
  - Output: Ended?
- **Version-specific requirements:** Limit node `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - If multiple items unexpectedly arrive, only a subset may continue
  - Default behavior should be verified in your n8n version
- **Sub-workflow reference:** None

#### 9. Ended?
- **Type and technical role:** `n8n-nodes-base.if`  
  Tests whether the Vapi call has completed.
- **Configuration choices:**
  - Compares `{{$json.status}}` to `ended`
  - True branch means finished
  - False branch means continue polling
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: Limit
  - True output: Voicemail?
  - False output: Polling
- **Version-specific requirements:** If node `typeVersion: 2.3`.
- **Edge cases or potential failure types:**
  - If Vapi uses different terminal states such as `completed`, `failed`, or `canceled`, they will not be treated as ended
  - Missing `status` field causes the comparison to fail
- **Sub-workflow reference:** None

#### 10. Polling
- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds a short delay between repeated status checks.
- **Configuration choices:**
  - Wait amount: `10`
  - Intended as a 10-second polling interval
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: Ended? (false branch)
  - Output: Get Call Details
- **Version-specific requirements:** Wait node `typeVersion: 1.1`.
- **Edge cases or potential failure types:**
  - Same unit ambiguity as the first Wait node
  - Poll loop has no explicit max retry count, so a stuck call could lead to very long-running executions
- **Sub-workflow reference:** None

---

## Block 5 — Call Outcome Classification and Google Sheets Logging

### Overview
This block determines whether the ended call reached voicemail or completed successfully, then appends the appropriate row to Google Sheets. Successful calls include structured outputs extracted from the Vapi response.

### Nodes Involved
- Voicemail?
- Log Voicemail
- Log Complete

### Node Details

#### 11. Voicemail?
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks the reason the call ended.
- **Configuration choices:**
  - Compares `{{$json.endedReason}}` to `voicemail`
  - True branch = voicemail
  - False branch = completed/other non-voicemail outcome
- **Key expressions or variables used:**
  - `={{ $json.endedReason }}`
- **Input and output connections:**
  - Input: Ended? (true branch)
  - True output: Log Voicemail
  - False output: Log Complete
- **Version-specific requirements:** If node `typeVersion: 2.3`.
- **Edge cases or potential failure types:**
  - Other end reasons like no-answer, busy, failed, canceled are treated as completed because only `voicemail` is separated
  - Missing `endedReason` may route incorrectly
- **Sub-workflow reference:** None

#### 12. Log Voicemail
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Logs voicemail outcomes into Google Sheets with fallback values.
- **Configuration choices:**
  - Operation: **append**
  - Mapping mode: **define below**
  - Uses data from the original form submission via node reference:
    - Name, Role, Email, Phone, Company, Request, Company Size
  - Writes fixed placeholders:
    - Budget: `N/A`
    - Intent?: `N/A`
    - Urgency: `N/A`
    - Motivation: `N/A`
    - Past Experience: `N/A`
    - Service Interest: `N/A`
  - Status: `Call Back`
- **Key expressions or variables used:**
  - `={{ $('On form submission').item.json.Name }}`
  - `={{ $('On form submission').item.json.Role }}`
  - `={{ $('On form submission').item.json.Email }}`
  - `={{ $('On form submission').item.json['Phone Number'] }}`
  - `={{ $('On form submission').item.json['Company Name'] }}`
  - `={{ $('On form submission').item.json.Request }}`
  - `={{ $('On form submission').item.json['Company Size'] }}`
- **Input and output connections:**
  - Input: Voicemail? (true branch)
  - Output: none
- **Version-specific requirements:** Google Sheets node `typeVersion: 4.7`.
- **Edge cases or potential failure types:**
  - Original form value for `Phone Number` may differ from the standardized value because this node references the trigger node, not the cleaned data node
  - Missing sheet headers or auth issues
  - If the workflow execution context changes, direct node references can become brittle
- **Sub-workflow reference:** None

#### 13. Log Complete
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Logs successfully completed qualification results, including structured outputs returned by Vapi.
- **Configuration choices:**
  - Operation: **append**
  - Mapping mode: **define below**
  - Uses original lead data from `On form submission`
  - Uses structured qualification results from `Limit`
  - Status: `Complete`
  - Structured outputs expected under:
    - `artifact.structuredOutputs['YOUR_BUDGET_UUID'].result`
    - `artifact.structuredOutputs['YOUR_INTENT_UUID'].result`
    - `artifact.structuredOutputs['YOUR_URGENCY_UUID'].result`
    - `artifact.structuredOutputs['YOUR_MOTIVATION_UUID'].result`
    - `artifact.structuredOutputs['YOUR_PAST_EXPERIENCE_UUID'].result`
    - `artifact.structuredOutputs['YOUR_SERVICE_INTEREST_UUID'].result`
- **Key expressions or variables used:**
  - Form data:
    - `={{ $('On form submission').item.json.Name }}`
    - `={{ $('On form submission').item.json.Role }}`
    - `={{ $('On form submission').item.json.Email }}`
    - `={{ $('On form submission').item.json['Phone Number'] }}`
    - `={{ $('On form submission').item.json['Company Name'] }}`
    - `={{ $('On form submission').item.json.Request }}`
    - `={{ $('On form submission').item.json['Company Size'] }}`
  - Vapi structured outputs:
    - `={{ $('Limit').item.json.artifact.structuredOutputs['YOUR_BUDGET_UUID'].result }}`
    - `={{ $('Limit').item.json.artifact.structuredOutputs['YOUR_INTENT_UUID'].result }}`
    - `={{ $('Limit').item.json.artifact.structuredOutputs['YOUR_URGENCY_UUID'].result }}`
    - `={{ $('Limit').item.json.artifact.structuredOutputs['YOUR_MOTIVATION_UUID'].result }}`
    - `={{ $('Limit').item.json.artifact.structuredOutputs['YOUR_PAST_EXPERIENCE_UUID'].result }}`
    - `={{ $('Limit').item.json.artifact.structuredOutputs['YOUR_SERVICE_INTEREST_UUID'].result }}`
- **Input and output connections:**
  - Input: Voicemail? (false branch)
  - Output: none
- **Version-specific requirements:** Google Sheets node `typeVersion: 4.7`.
- **Edge cases or potential failure types:**
  - Placeholder UUIDs must be replaced with real Vapi structured output IDs
  - If any structured output is absent, the expression can fail or return undefined
  - Same note as voicemail logging: phone value is pulled from original submission, not standardized data
  - Sheets auth/header issues
- **Sub-workflow reference:** None

---

## Block 6 — Embedded Documentation Sticky Notes

### Overview
The workflow includes multiple sticky notes that document purpose, setup, requirements, and customization guidance directly inside the canvas. These are not executable but are important for maintenance and reproduction.

### Nodes Involved
- Sticky Note Main
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### 14. Sticky Note Main
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for the full workflow.
- **Configuration choices:** Large note summarizing who the workflow is for, what it does, setup steps, requirements, and customization ideas.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:** None functionally
- **Sub-workflow reference:** None

#### 15. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the form and phone validation stage.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:** None functionally
- **Sub-workflow reference:** None

#### 16. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the Vapi calling stage.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:** None functionally
- **Sub-workflow reference:** None

#### 17. Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents wait and polling behavior.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:** None functionally
- **Sub-workflow reference:** None

#### 18. Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the logging stage, required structured outputs, and required Google Sheet headers.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:** None functionally
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Receives inbound lead data from hosted form |  | Standardize Data | ## 1. Form & Phone Validation<br><br>The **On form submission** node captures lead details via a web form. The **Standardize Data** node cleans the phone input to a 10-digit US format. Invalid numbers get logged to Google Sheets and skipped.<br><br>**Setup:** Customize the form fields and title to match your business. |
| Standardize Data | Code | Cleans and validates phone numbers | On form submission | Incorrect Phone? | ## 1. Form & Phone Validation<br><br>The **On form submission** node captures lead details via a web form. The **Standardize Data** node cleans the phone input to a 10-digit US format. Invalid numbers get logged to Google Sheets and skipped.<br><br>**Setup:** Customize the form fields and title to match your business. |
| Incorrect Phone? | If | Branches invalid phone numbers away from call flow | Standardize Data | Log Incorrect Phone; Call Lead | ## 1. Form & Phone Validation<br><br>The **On form submission** node captures lead details via a web form. The **Standardize Data** node cleans the phone input to a 10-digit US format. Invalid numbers get logged to Google Sheets and skipped.<br><br>**Setup:** Customize the form fields and title to match your business. |
| Log Incorrect Phone | Google Sheets | Logs leads with invalid phone numbers | Incorrect Phone? |  | ## 2. AI Voice Call<br><br>Valid leads get an instant phone call from your Vapi AI assistant. The assistant qualifies them through a natural conversation, gathering structured data about their needs.<br><br>**Setup:** Update the Vapi assistant ID and phone number ID in the **Call Lead** node JSON body. |
| Call Lead | HTTP Request | Starts outbound Vapi AI qualification call | Incorrect Phone? | Wait | ## 2. AI Voice Call<br><br>Valid leads get an instant phone call from your Vapi AI assistant. The assistant qualifies them through a natural conversation, gathering structured data about their needs.<br><br>**Setup:** Update the Vapi assistant ID and phone number ID in the **Call Lead** node JSON body. |
| Wait | Wait | Initial delay before checking call status | Call Lead | Get Call Details | ## 3. Call Monitoring<br><br>The workflow waits 60 seconds for the call to complete, then polls the Vapi API every 10 seconds until the call status is "ended". This ensures no results are lost even for longer conversations.<br><br>**Customize:** Adjust the initial wait time in **Wait** if your calls are typically shorter or longer. |
| Get Call Details | HTTP Request | Retrieves current Vapi call status and artifacts | Wait; Polling | Limit | ## 3. Call Monitoring<br><br>The workflow waits 60 seconds for the call to complete, then polls the Vapi API every 10 seconds until the call status is "ended". This ensures no results are lost even for longer conversations.<br><br>**Customize:** Adjust the initial wait time in **Wait** if your calls are typically shorter or longer. |
| Limit | Limit | Restricts downstream processing to limited item count | Get Call Details | Ended? | ## 3. Call Monitoring<br><br>The workflow waits 60 seconds for the call to complete, then polls the Vapi API every 10 seconds until the call status is "ended". This ensures no results are lost even for longer conversations.<br><br>**Customize:** Adjust the initial wait time in **Wait** if your calls are typically shorter or longer. |
| Ended? | If | Determines whether call has finished | Limit | Voicemail?; Polling | ## 3. Call Monitoring<br><br>The workflow waits 60 seconds for the call to complete, then polls the Vapi API every 10 seconds until the call status is "ended". This ensures no results are lost even for longer conversations.<br><br>**Customize:** Adjust the initial wait time in **Wait** if your calls are typically shorter or longer. |
| Polling | Wait | Delay between repeated status checks | Ended? | Get Call Details | ## 4. Results Logging<br><br>Completed calls are logged with AI-extracted insights: service interest, motivation, urgency, past experience, budget, and intent. Voicemails get logged separately with a "Call Back" status.<br><br>**Setup:** Replace the structured output UUIDs in **Log Complete** with the actual UUIDs from your Vapi assistant configuration.<br><br>**Google Sheet headers:**<br>`Date` · `Name` · `Phone` · `Email` · `Company` · `Role` · `Request` · `Company Size` · `Service Interest` · `Motivation` · `Urgency` · `Past Experience` · `Budget` · `Intent?` · `Status` |
| Voicemail? | If | Separates voicemail outcomes from completed conversations | Ended? | Log Voicemail; Log Complete | ## 4. Results Logging<br><br>Completed calls are logged with AI-extracted insights: service interest, motivation, urgency, past experience, budget, and intent. Voicemails get logged separately with a "Call Back" status.<br><br>**Setup:** Replace the structured output UUIDs in **Log Complete** with the actual UUIDs from your Vapi assistant configuration.<br><br>**Google Sheet headers:**<br>`Date` · `Name` · `Phone` · `Email` · `Company` · `Role` · `Request` · `Company Size` · `Service Interest` · `Motivation` · `Urgency` · `Past Experience` · `Budget` · `Intent?` · `Status` |
| Log Voicemail | Google Sheets | Appends voicemail outcomes for callback | Voicemail? |  | ## 4. Results Logging<br><br>Completed calls are logged with AI-extracted insights: service interest, motivation, urgency, past experience, budget, and intent. Voicemails get logged separately with a "Call Back" status.<br><br>**Setup:** Replace the structured output UUIDs in **Log Complete** with the actual UUIDs from your Vapi assistant configuration.<br><br>**Google Sheet headers:**<br>`Date` · `Name` · `Phone` · `Email` · `Company` · `Role` · `Request` · `Company Size` · `Service Interest` · `Motivation` · `Urgency` · `Past Experience` · `Budget` · `Intent?` · `Status` |
| Log Complete | Google Sheets | Appends completed qualification results with structured outputs | Voicemail? |  | ## 4. Results Logging<br><br>Completed calls are logged with AI-extracted insights: service interest, motivation, urgency, past experience, budget, and intent. Voicemails get logged separately with a "Call Back" status.<br><br>**Setup:** Replace the structured output UUIDs in **Log Complete** with the actual UUIDs from your Vapi assistant configuration.<br><br>**Google Sheet headers:**<br>`Date` · `Name` · `Phone` · `Email` · `Company` · `Role` · `Request` · `Company Size` · `Service Interest` · `Motivation` · `Urgency` · `Past Experience` · `Budget` · `Intent?` · `Status` |
| Sticky Note Main | Sticky Note | Embedded workflow documentation |  |  | ## Qualify Leads with AI Voice Calls Using Vapi and Log to Google Sheets<br><br>### Who is this for<br>Agencies, sales teams, and service businesses who want to instantly qualify inbound leads with an AI-powered phone call instead of manual follow-up.<br><br>### What it does<br>1. Captures lead info via a web form (name, phone, email, company, role, request)<br>2. Validates the phone number format<br>3. Triggers an AI voice call via Vapi to qualify the lead<br>4. Waits for the call to complete, polling until finished<br>5. Logs results to Google Sheets — including AI-extracted insights like service interest, motivation, urgency, budget, and intent<br><br>### How to set up<br>1. Create a Vapi account and set up a voice assistant with structured outputs for: Service Interest, Motivation, Urgency, Past Experience, Budget, Intent<br>2. Copy the Google Sheet template or create one with these headers:<br>`Date` · `Name` · `Phone` · `Email` · `Company` · `Role` · `Request` · `Company Size` · `Service Interest` · `Motivation` · `Urgency` · `Past Experience` · `Budget` · `Intent?` · `Status`<br>3. Update the **Call Lead** node with your Vapi assistant ID and phone number ID<br>4. Update the **Log Complete** node with your Vapi structured output UUIDs<br>5. Paste your Google Sheet URL into all three Google Sheets nodes<br>6. Connect your Bearer Auth (Vapi API key) and Google Sheets credentials<br>7. Activate the workflow and share the form URL<br><br>### Requirements<br>- n8n account (cloud or self-hosted)<br>- Vapi account with a configured voice assistant and phone number<br>- Google Sheets (one sheet with the column headers above)<br><br>### How to customize<br>- Edit the form fields to match your intake needs<br>- Adjust the Vapi assistant prompt for your industry<br>- Change the wait time (default 60s) based on typical call length<br>- Add email notifications when high-intent leads are detected |
| Sticky Note2 | Sticky Note | Embedded note for form and validation stage |  |  | ## 1. Form & Phone Validation<br><br>The **On form submission** node captures lead details via a web form. The **Standardize Data** node cleans the phone input to a 10-digit US format. Invalid numbers get logged to Google Sheets and skipped.<br><br>**Setup:** Customize the form fields and title to match your business. |
| Sticky Note3 | Sticky Note | Embedded note for AI call stage |  |  | ## 2. AI Voice Call<br><br>Valid leads get an instant phone call from your Vapi AI assistant. The assistant qualifies them through a natural conversation, gathering structured data about their needs.<br><br>**Setup:** Update the Vapi assistant ID and phone number ID in the **Call Lead** node JSON body. |
| Sticky Note4 | Sticky Note | Embedded note for polling stage |  |  | ## 3. Call Monitoring<br><br>The workflow waits 60 seconds for the call to complete, then polls the Vapi API every 10 seconds until the call status is "ended". This ensures no results are lost even for longer conversations.<br><br>**Customize:** Adjust the initial wait time in **Wait** if your calls are typically shorter or longer. |
| Sticky Note5 | Sticky Note | Embedded note for logging stage |  |  | ## 4. Results Logging<br><br>Completed calls are logged with AI-extracted insights: service interest, motivation, urgency, past experience, budget, and intent. Voicemails get logged separately with a "Call Back" status.<br><br>**Setup:** Replace the structured output UUIDs in **Log Complete** with the actual UUIDs from your Vapi assistant configuration.<br><br>**Google Sheet headers:**<br>`Date` · `Name` · `Phone` · `Email` · `Company` · `Role` · `Request` · `Company Size` · `Service Interest` · `Motivation` · `Urgency` · `Past Experience` · `Budget` · `Intent?` · `Status` |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: **Qualify leads with AI voice calls using Vapi and log to Google Sheets**.

2. **Add a Form Trigger node** named **On form submission**.
   - Set form title to: `Work with us!`
   - Set description to: `Fill out the form and we will reach out ASAP.`
   - Add these required fields:
     1. `Name` — text
     2. `Phone Number` — number
     3. `Email` — email
     4. `Company Name` — text
     5. `Role` — text
     6. `Request` — text
     7. `Company Size` — dropdown with options:
        - `1`
        - `2-10`
        - `11-50`
        - `51-100`
        - `101+`

3. **Add a Code node** named **Standardize Data** and connect:
   - `On form submission` → `Standardize Data`

4. **Paste this logic into Standardize Data** in JavaScript form:
   - Read all input items
   - Convert `Phone Number` to string
   - Remove non-digits
   - If length is 10, keep it
   - If length is 11 and begins with `1`, remove the leading `1`
   - Otherwise set phone to `incorrect format`
   - Return the modified items
   - The code behavior should preserve all other fields unchanged

5. **Add an If node** named **Incorrect Phone?** and connect:
   - `Standardize Data` → `Incorrect Phone?`
   - Configure one condition:
     - Left value: `{{$json["Phone Number"]}}`
     - Operator: `equals`
     - Right value: `incorrect format`

6. **Add a Google Sheets node** named **Log Incorrect Phone**.
   - Connect:
     - `Incorrect Phone?` true output → `Log Incorrect Phone`
   - Set operation to **Append**
   - Connect Google Sheets credentials
   - Select or paste your target spreadsheet
   - Select the target sheet/tab
   - Map columns manually:
     - `Date` → `{{$now.format('yyyy-MM-dd hh:mm a')}}`
     - `Name` → `{{$json.Name}}`
     - `Role` → `{{$json.Role}}`
     - `Email` → `{{$json.Email}}`
     - `Phone` → `{{$json['Phone Number']}}`
     - `Status` → `Incorrect Phone #`
     - `Company` → `{{$json['Company Name']}}`
     - `Request` → `{{$json.Request}}`
     - `Company Size` → `{{$json['Company Size']}}`

7. **Prepare your Google Sheet** before building the remaining nodes.
   - Create one spreadsheet and one worksheet/tab
   - Add these headers:
     - `Date`
     - `Name`
     - `Phone`
     - `Email`
     - `Company`
     - `Role`
     - `Request`
     - `Company Size`
     - `Service Interest`
     - `Motivation`
     - `Urgency`
     - `Past Experience`
     - `Budget`
     - `Intent?`
     - `Status`

8. **Set up Vapi before the call node.**
   - Create a Vapi account
   - Create a voice assistant
   - Assign a phone number
   - Configure structured outputs for:
     - Service Interest
     - Motivation
     - Urgency
     - Past Experience
     - Budget
     - Intent
   - Copy:
     - your Vapi API key
     - your `assistantId`
     - your `phoneNumberId`
     - each structured output UUID

9. **Create an HTTP Bearer Auth credential** in n8n for Vapi.
   - Use your Vapi API key as bearer token.
   - You will attach this credential to both Vapi HTTP Request nodes.

10. **Add an HTTP Request node** named **Call Lead**.
    - Connect:
      - `Incorrect Phone?` false output → `Call Lead`
    - Configure:
      - Method: `POST`
      - URL: `https://api.vapi.ai/call`
      - Authentication: **Generic Credential Type**
      - Generic Auth Type: **HTTP Bearer Auth**
      - Select your Vapi bearer credential
      - Send body: enabled
      - Body type: JSON
    - Build the JSON body with:
      - `assistantId`: your real assistant ID
      - `phoneNumberId`: your real Vapi phone number ID
      - `customers`: array with one object containing `number` as `+1{{$json['Phone Number']}}`
      - `assistantOverrides.variableValues` with:
        - `lead_name` = `{{$json.Name}}`
        - `lead_company_name` = `{{$json['Company Name']}}`
        - `lead_request` = `{{$json.Request}}`

11. **Add a Wait node** named **Wait**.
    - Connect:
      - `Call Lead` → `Wait`
    - Configure it as a delay of **60 seconds**.

12. **Add an HTTP Request node** named **Get Call Details**.
    - Connect:
      - `Wait` → `Get Call Details`
    - Configure:
      - Method: `GET`
      - URL: `=https://api.vapi.ai/call/{{ $json.results[0].id }}`
      - Authentication: Generic Credential Type
      - Generic Auth Type: HTTP Bearer Auth
      - Reuse the Vapi bearer credential
    - Important: test the actual response from **Call Lead**.
      - If your Vapi response returns a different structure, update the URL expression accordingly.
      - Many implementations may return `id` directly rather than `results[0].id`.

13. **Add a Limit node** named **Limit**.
    - Connect:
      - `Get Call Details` → `Limit`
    - Leave default settings unless you want explicit one-item behavior.
    - If your version requires it, set it to pass only 1 item.

14. **Add an If node** named **Ended?**.
    - Connect:
      - `Limit` → `Ended?`
    - Configure condition:
      - Left value: `{{$json.status}}`
      - Operator: `equals`
      - Right value: `ended`

15. **Add a second Wait node** named **Polling**.
    - Connect:
      - `Ended?` false output → `Polling`
      - `Polling` → `Get Call Details`
    - Set delay to **10 seconds**
   - This creates the polling loop.

16. **Add another If node** named **Voicemail?**.
    - Connect:
      - `Ended?` true output → `Voicemail?`
    - Configure condition:
      - Left value: `{{$json.endedReason}}`
      - Operator: `equals`
      - Right value: `voicemail`

17. **Add a Google Sheets node** named **Log Voicemail**.
    - Connect:
      - `Voicemail?` true output → `Log Voicemail`
    - Set operation to **Append**
    - Use the same Google Sheets credential and same sheet/tab
    - Map fields:
      - `Date` → `{{$now.format('yyyy-MM-dd hh:mm a')}}`
      - `Name` → `{{$('On form submission').item.json.Name}}`
      - `Role` → `{{$('On form submission').item.json.Role}}`
      - `Email` → `{{$('On form submission').item.json.Email}}`
      - `Phone` → `{{$('On form submission').item.json['Phone Number']}}`
      - `Budget` → `N/A`
      - `Status` → `Call Back`
      - `Company` → `{{$('On form submission').item.json['Company Name']}}`
      - `Intent?` → `N/A`
      - `Request` → `{{$('On form submission').item.json.Request}}`
      - `Urgency` → `N/A`
      - `Motivation` → `N/A`
      - `Company Size` → `{{$('On form submission').item.json['Company Size']}}`
      - `Past Experience` → `N/A`
      - `Service Interest` → `N/A`

18. **Add a Google Sheets node** named **Log Complete**.
    - Connect:
      - `Voicemail?` false output → `Log Complete`
    - Set operation to **Append**
    - Use the same spreadsheet and same tab
    - Map fields:
      - `Date` → `{{$now.format('yyyy-MM-dd hh:mm a')}}`
      - `Name` → `{{$('On form submission').item.json.Name}}`
      - `Role` → `{{$('On form submission').item.json.Role}}`
      - `Email` → `{{$('On form submission').item.json.Email}}`
      - `Phone` → `{{$('On form submission').item.json['Phone Number']}}`
      - `Budget` → `{{$('Limit').item.json.artifact.structuredOutputs['YOUR_BUDGET_UUID'].result}}`
      - `Status` → `Complete`
      - `Company` → `{{$('On form submission').item.json['Company Name']}}`
      - `Intent?` → `{{$('Limit').item.json.artifact.structuredOutputs['YOUR_INTENT_UUID'].result}}`
      - `Request` → `{{$('On form submission').item.json.Request}}`
      - `Urgency` → `{{$('Limit').item.json.artifact.structuredOutputs['YOUR_URGENCY_UUID'].result}}`
      - `Motivation` → `{{$('Limit').item.json.artifact.structuredOutputs['YOUR_MOTIVATION_UUID'].result}}`
      - `Company Size` → `{{$('On form submission').item.json['Company Size']}}`
      - `Past Experience` → `{{$('Limit').item.json.artifact.structuredOutputs['YOUR_PAST_EXPERIENCE_UUID'].result}}`
      - `Service Interest` → `{{$('Limit').item.json.artifact.structuredOutputs['YOUR_SERVICE_INTEREST_UUID'].result}}`
    - Replace all placeholder UUIDs with real UUIDs from your Vapi assistant configuration.

19. **Test the workflow with three scenarios.**
    - Scenario A: invalid phone number
      - Confirm only **Log Incorrect Phone** runs
    - Scenario B: voicemail
      - Confirm **Log Voicemail** receives a row with `Call Back`
    - Scenario C: completed qualification call
      - Confirm **Log Complete** receives structured output values

20. **Verify the Vapi payload structure carefully.**
    - If `Call Lead` does not return `results[0].id`, store the returned call ID explicitly before polling.
    - A safer variant is to insert a Set or Code node after `Call Lead` to normalize the call ID into a known field like `callId`, then use `https://api.vapi.ai/call/{{$json.callId}}`.

21. **Consider improving robustness before production.**
    - Add a max polling count to avoid infinite loops
    - Handle non-voicemail failure states like:
      - busy
      - no-answer
      - failed
      - canceled
    - Reference standardized phone output instead of original form value in logging nodes if you want normalized phone storage
    - Add alerts for high-intent leads

22. **Activate the workflow** and share the form URL generated by the Form Trigger node.

### Credential configuration summary
- **Google Sheets credentials**
  - Required for:
    - Log Incorrect Phone
    - Log Voicemail
    - Log Complete
- **HTTP Bearer Auth credentials**
  - Required for:
    - Call Lead
    - Get Call Details
  - Token value: your Vapi API key

### Sub-workflow setup
- This workflow does **not** use any sub-workflows.
- There are **no additional workflow entry points** besides the Form Trigger.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Agencies, sales teams, and service businesses who want to instantly qualify inbound leads with an AI-powered phone call instead of manual follow-up. | Workflow purpose |
| Create a Vapi account and set up a voice assistant with structured outputs for: Service Interest, Motivation, Urgency, Past Experience, Budget, Intent. | Vapi setup |
| Use one Google Sheet with headers: `Date`, `Name`, `Phone`, `Email`, `Company`, `Role`, `Request`, `Company Size`, `Service Interest`, `Motivation`, `Urgency`, `Past Experience`, `Budget`, `Intent?`, `Status`. | Google Sheets setup |
| Update the **Call Lead** node with your Vapi assistant ID and phone number ID. | Vapi configuration |
| Update the **Log Complete** node with your Vapi structured output UUIDs. | Vapi structured outputs mapping |
| Paste your Google Sheet URL into all three Google Sheets nodes. | Google Sheets node configuration |
| Connect Bearer Auth for Vapi API access and Google Sheets credentials before activation. | Credentials |
| Customize the form fields, Vapi assistant prompt, wait duration, and add notifications for high-intent leads if needed. | Customization guidance |

## Additional implementation observations
- The workflow is **US-phone-number-specific** in its current form.
- The `Phone Number` logged in voicemail and complete branches comes from the original form submission, not the standardized value.
- The polling loop has **no explicit termination limit**, so production use would benefit from a retry cap or timeout branch.
- The `Get Call Details` node assumes the call creation response contains `results[0].id`; verify this against your actual Vapi API response before relying on it.