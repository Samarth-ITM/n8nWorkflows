Populate Retell Dynamic Variables with Google Sheets Data for Call Handling

https://n8nworkflows.xyz/workflows/populate-retell-dynamic-variables-with-google-sheets-data-for-call-handling-3385


# Populate Retell Dynamic Variables with Google Sheets Data for Call Handling

### 1. Workflow Overview

This workflow is designed to integrate Retell Voice Agents with user data stored in Google Sheets, enabling dynamic personalization of voice call interactions. It listens for inbound webhook calls from Retell, filters them by IP address for security, retrieves user information based on the caller’s phone number from a Google Sheet, and returns this data formatted as dynamic variables for Retell to use in its voice prompts.

**Target Use Cases:**  
- Builders of Retell Voice Agents who want to personalize call handling by injecting user-specific data dynamically.  
- Scenarios where user data is maintained in Google Sheets and needs to be fetched in real-time during inbound calls.

**Logical Blocks:**  
- **1.1 Input Reception and Security Filtering:** Receives webhook calls from Retell and filters by IP.  
- **1.2 User Data Retrieval:** Queries Google Sheets to find the user record matching the caller’s phone number.  
- **1.3 Response Formatting and Delivery:** Formats the retrieved data into Retell’s expected dynamic variable structure and responds to the webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Security Filtering

**Overview:**  
This block receives inbound HTTP POST requests from Retell’s webhook and ensures only requests from Retell’s whitelisted IP address are processed.

**Nodes Involved:**  
- Webhook

**Node Details:**  

- **Webhook**  
  - *Type & Role:* HTTP Webhook node; entry point for inbound Retell webhook calls.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: `retell-dynamic-variables` (customizable)  
    - IP Whitelist: `100.20.5.228` (Retell’s IP to restrict access)  
    - Response Mode: Uses a dedicated response node to send replies.  
  - *Key Expressions:* None (static configuration)  
  - *Connections:* Output connects to "Get user in DB by Phone Number" node.  
  - *Version Requirements:* n8n version supporting IP whitelisting on webhook nodes (v0.152.0+ recommended).  
  - *Potential Failures:*  
    - Requests from non-whitelisted IPs are rejected (no processing).  
    - Incorrect HTTP method or path results in 404 or method not allowed.  
    - Network or server downtime may cause webhook unavailability.  

---

#### 2.2 User Data Retrieval

**Overview:**  
This block queries a Google Sheet to find a user record matching the incoming phone number extracted from the webhook payload.

**Nodes Involved:**  
- Get user in DB by Phone Number

**Node Details:**  

- **Get user in DB by Phone Number**  
  - *Type & Role:* Google Sheets node; performs a lookup to retrieve user data.  
  - *Configuration:*  
    - Document ID: Google Sheet containing user data (example sheet provided).  
    - Sheet Name: First sheet (`gid=0`), named "Users".  
    - Filter: Looks up the "Phone Number" column for a value matching `{{$json.body.call_inbound.from_number}}` from the webhook payload.  
  - *Key Expressions:*  
    - Lookup value expression: `={{ $json.body.call_inbound.from_number }}`  
  - *Connections:* Input from Webhook node; output to Respond to Webhook node.  
  - *Credentials:* Uses OAuth2 credentials for Google Sheets API access.  
  - *Version Requirements:* Google Sheets node v4.5+ recommended for filter UI and OAuth2 support.  
  - *Potential Failures:*  
    - Authentication errors if OAuth2 credentials expire or are invalid.  
    - No matching phone number found results in empty output (may cause downstream issues if not handled).  
    - Google API rate limits or downtime.  
    - Phone number format mismatches (must start with '+', no spaces).  

---

#### 2.3 Response Formatting and Delivery

**Overview:**  
This block formats the retrieved user data into Retell’s dynamic variable JSON structure and sends it back as the webhook response.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**  

- **Respond to Webhook**  
  - *Type & Role:* Respond to Webhook node; sends the final JSON response back to Retell.  
  - *Configuration:*  
    - Respond with: JSON  
    - Response Body: A JSON object embedding dynamic variables such as `first_name`, `last_name`, `email`, and two user variables, each mapped to corresponding Google Sheets columns using expressions like `{{ $json['First Name'] }}`.  
  - *Key Expressions:*  
    - Uses templated expressions to extract fields from the Google Sheets row, e.g., `{{ $json['First Name'] }}`, `{{ $json['User Variable 1'] }}`.  
  - *Connections:* Input from "Get user in DB by Phone Number" node; no outputs.  
  - *Version Requirements:* Respond to Webhook node v1.1+ for JSON response support.  
  - *Potential Failures:*  
    - If input data is empty (no user found), variables will be undefined or empty strings.  
    - Expression evaluation errors if column names change or are missing.  
    - Malformed JSON response if expressions fail.  

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                          | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                   |
|-----------------------------|----------------------------|----------------------------------------|----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| Webhook                     | Webhook                    | Receives inbound webhook from Retell   |                            | Get user in DB by Phone Number | Change the path if needed                                                                     |
| Get user in DB by Phone Number | Google Sheets             | Retrieves user data by phone number    | Webhook                    | Respond to Webhook             | Replace with your own Google Sheets, including the dynamic variables of your Retell Agent     |
| Respond to Webhook           | Respond to Webhook          | Sends formatted dynamic variables back | Get user in DB by Phone Number |                               | Adapt the response to match your Retell dynamic variables                                    |
| Sticky Note1                 | Sticky Note                | Documentation and overview notes       |                            |                               | ## Handle Retell's Inbound call webhooks (full detailed description and instructions)        |
| Sticky Note3                 | Sticky Note                | Instruction for Webhook node path      |                            |                               | Change the path if needed                                                                     |
| Sticky Note                  | Sticky Note                | Instruction for Google Sheets node     |                            |                               | Replace with your own Google Sheets, including the dynamic variables of your Retell Agent     |
| Sticky Note2                 | Sticky Note                | Instruction for Respond node            |                            |                               | Adapt the response to match your Retell dynamic variables                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Add a **Webhook** node.  
   - Set HTTP Method to **POST**.  
   - Set Path to `retell-dynamic-variables` (or your preferred path).  
   - Enable **IP Whitelist** and add `100.20.5.228` to restrict access to Retell’s IP.  
   - Set Response Mode to **Respond to Webhook** (this allows a separate node to send the response).  

2. **Create the Google Sheets Node**  
   - Add a **Google Sheets** node named "Get user in DB by Phone Number".  
   - Connect the Webhook node’s output to this node’s input.  
   - Configure credentials with your Google Sheets OAuth2 account.  
   - Set Document ID to your Google Sheet containing user data.  
   - Set Sheet Name to the sheet containing user records (e.g., "Users").  
   - Under Filters, add a filter:  
     - Lookup Column: `Phone Number`  
     - Lookup Value: Expression `={{ $json.body.call_inbound.from_number }}`  
   - This will search for the row where the phone number matches the incoming call number.  

3. **Create the Respond to Webhook Node**  
   - Add a **Respond to Webhook** node.  
   - Connect the Google Sheets node output to this node input.  
   - Set **Respond With** to **JSON**.  
   - In the Response Body, enter the following JSON template, adapting variable names to your sheet’s columns if needed:  
     ```json
     {
       "call_inbound": {
         "dynamic_variables": {
           "first_name": "{{ $json['First Name'] }}",
           "last_name": "{{ $json['Last name'] }}",
           "email": "{{ $json['E-Mail'] }}",
           "variable_1": "{{ $json['User Variable 1'] }}",
           "variable_2": "{{ $json['User Variable 2'] }}"
         },
         "metadata": {}
       }
     }
     ```  
   - This formats the response to Retell’s expected dynamic variable structure.  

4. **Connect Nodes**  
   - Connect the Webhook node’s main output to the Google Sheets node input.  
   - Connect the Google Sheets node’s main output to the Respond to Webhook node input.  

5. **Set Up Credentials**  
   - For Google Sheets node, configure OAuth2 credentials with access to the target spreadsheet.  
   - Ensure the Google Sheet has a column named exactly `Phone Number` with phone numbers formatted starting with `+` and no spaces.  

6. **Deploy and Test**  
   - Activate the workflow.  
   - Copy the webhook URL from the Webhook node (e.g., `https://your-instance.app.n8n.cloud/webhook/retell-dynamic-variables`).  
   - In your Retell account, configure your phone number to use this webhook URL for inbound calls.  
   - Make a test call from a phone number listed in your Google Sheet to verify dynamic variables are correctly populated.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                    | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Retell is a platform to create Voice Agents that handle calls based on prompts or conversational flows.                                                                                                                                                                         | https://www.retellai.com/                                                                          |
| Dynamic variables in Retell prompts use the syntax `{{variable_name}}` and are replaced by values returned from this webhook integration.                                                                                                                                       | https://docs.retellai.com/build/dynamic-variables                                                 |
| The Google Sheet must have phone numbers formatted with a leading `+` and no spaces, matching the incoming call number format exactly.                                                                                                                                          | See prerequisites in workflow description                                                         |
| You can replace Google Sheets with any other database or data source, provided the phone number lookup and response formatting are adapted accordingly.                                                                                                                          | Workflow is adaptable to other data sources                                                       |
| Retell’s inbound webhook IP address is whitelisted here for security; update if Retell changes their IPs.                                                                                                                                                                        | IP whitelist in Webhook node configuration                                                        |
| For detailed Retell inbound webhook documentation, see: https://docs.retellai.com/features/inbound-call-webhook                                                                                                                                                                 | Retell documentation link                                                                         |
| Example Google Sheet template for user data: https://docs.google.com/spreadsheets/d/1TYgk8PK5w2l8Q5NtepdyLvgtuHXBHcODy-2hXOPP6AU/edit?usp=sharing                                                                                                                                  | Sample data source                                                                                |

---

This documentation fully describes the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.