Qualify Meta Ads Leads with WhatsApp Verification, Gemini AI & Zoho CRM

https://n8nworkflows.xyz/workflows/qualify-meta-ads-leads-with-whatsapp-verification--gemini-ai---zoho-crm-6529


# Qualify Meta Ads Leads with WhatsApp Verification, Gemini AI & Zoho CRM

### 1. Workflow Overview

This workflow automates the qualification of leads generated via Meta (Facebook) Ads by integrating WhatsApp verification, AI response classification using Google Gemini, and contact management in Zoho CRM. It is designed for marketing and sales teams needing to verify lead interest and update CRM records accordingly, minimizing manual follow-up and improving lead quality.

The workflow logically splits into the following blocks:

- **1.1 Scheduled Lead Retrieval:** Periodically fetches new leads from Facebook Lead Ads.
- **1.2 Lead Data Extraction & CRM Lookup:** Extracts contact details from the leads and checks for existing records in Zoho CRM.
- **1.3 New Lead Creation & WhatsApp Confirmation:** Creates new contacts in Zoho CRM and sends WhatsApp messages to confirm interest.
- **1.4 WhatsApp Response Reception:** Receives WhatsApp replies through a webhook.
- **1.5 AI Classification of Responses:** Uses Google Gemini LLM to classify WhatsApp replies into categories (confirmed, declined, human requested, invalid).
- **1.6 Response Routing and CRM Update:** Routes responses, sends appropriate WhatsApp replies, and updates Zoho CRM contact status and ownership accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Lead Retrieval

- **Overview:** Periodically triggers the workflow to fetch new leads from Facebook Lead Ads.
- **Nodes Involved:**  
  - Trigger: Schedule Lead Check  
  - Fetch FB Leads

- **Node Details:**

  - **Trigger: Schedule Lead Check**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at specific intervals (every few seconds) to check for new leads automatically.  
    - Configuration: Interval trigger set to run every few seconds (default seconds interval).  
    - Inputs: None  
    - Outputs: Triggers "Fetch FB Leads" node.  
    - Edge Cases: Excessive triggering can lead to API rate limits; recommended to adjust interval based on expected lead volume.  

  - **Fetch FB Leads**  
    - Type: HTTP Request  
    - Role: Calls Facebook Graph API to retrieve new leads from a specific Lead Ad Form.  
    - Configuration: URL uses Facebook Graph API endpoint with a dynamic access token (`{{FACEBOOK_ACCESS_TOKEN}}`).  
    - Inputs: Trigger node output.  
    - Outputs: Passes fetched JSON lead data to "Extract Lead Info".  
    - Edge Cases: Possible failures include expired or invalid Facebook Access Token, API rate limits, or no new leads returned. Handle empty data gracefully.

---

#### 2.2 Lead Data Extraction & CRM Lookup

- **Overview:** Extracts key lead details (Name, Phone, Email) and checks if the contact already exists in Zoho CRM.
- **Nodes Involved:**  
  - Extract Lead Info  
  - Check Zoho Contact Exists  
  - If New Lead

- **Node Details:**

  - **Extract Lead Info**  
    - Type: Set  
    - Role: Parses the Facebook Lead Ads JSON to extract the first lead's Name, Phone Number, and Email into simplified fields for downstream use.  
    - Configuration: Uses expressions to extract fields from nested JSON arrays (e.g., `$json.data[0].field_data[0].values[0]` for Name).  
    - Inputs: Output of "Fetch FB Leads".  
    - Outputs: Passes extracted data to "Check Zoho Contact Exists".  
    - Edge Cases: If no lead data is present or fields are missing, may cause empty or invalid assignments.

  - **Check Zoho Contact Exists**  
    - Type: HTTP Request  
    - Role: Searches Zoho CRM Contacts by phone number to determine if lead already exists.  
    - Configuration: Uses Zoho CRM API v2 search endpoint with OAuth2 authentication; phone number passed as query parameter.  
    - Inputs: Extracted lead info.  
    - Outputs: Passes search result to "If New Lead".  
    - Edge Cases: Network errors, expired OAuth2 token, or no matching contacts found.

  - **If New Lead**  
    - Type: If  
    - Role: Checks if the Zoho search returned any data; routes workflow based on whether this is a new lead or an existing contact.  
    - Configuration: Condition checks if `$json.data` is empty or not.  
    - Inputs: Output of "Check Zoho Contact Exists".  
    - Outputs: If new lead, triggers lead creation and WhatsApp confirmation; else stops or continues as designed.  
    - Edge Cases: Incorrect condition logic could misclassify leads.

---

#### 2.3 New Lead Creation & WhatsApp Confirmation

- **Overview:** Creates a new contact in Zoho CRM with "Pending" status and sends a WhatsApp message asking the lead to confirm their interest.
- **Nodes Involved:**  
  - Create Zoho Contact  
  - Send WhatsApp Confirmation

- **Node Details:**

  - **Create Zoho Contact**  
    - Type: Zoho CRM  
    - Role: Creates a new contact record with extracted lead info and sets a custom "Status" field to "Pending".  
    - Configuration: Maps Name, Email, Phone, and custom "Status" field.  
    - Inputs: New lead data flow from "If New Lead".  
    - Outputs: Passes to "Send WhatsApp Confirmation".  
    - Edge Cases: API authentication failures, invalid data formats, or duplicate entries if checks fail.

  - **Send WhatsApp Confirmation**  
    - Type: HTTP Request  
    - Role: Sends a WhatsApp message via Twilio API to the lead’s phone number asking for confirmation ("Hi [Name], confirm yes or no").  
    - Configuration: Uses Twilio WhatsApp sandbox number as sender, dynamic "To" number from lead phone, HTTP Basic Auth credentials stored securely.  
    - Inputs: Lead contact info from "Create Zoho Contact".  
    - Outputs: No direct downstream outputs; awaits response via webhook.  
    - Edge Cases: Invalid phone numbers, Twilio account limits, message delivery failure.

---

#### 2.4 WhatsApp Response Reception

- **Overview:** Receives WhatsApp replies from leads via Twilio webhook integration.
- **Nodes Involved:**  
  - Receive WhatsApp Response

- **Node Details:**

  - **Receive WhatsApp Response**  
    - Type: Webhook  
    - Role: Receives incoming POST requests from Twilio when a lead replies on WhatsApp.  
    - Configuration: Webhook path set to "custrep", HTTP POST method.  
    - Inputs: External Twilio webhook calls.  
    - Outputs: Passes message content to "Classify Response (AI)".  
    - Edge Cases: Webhook URL must be publicly accessible and configured in Twilio. Payload format changes may break parsing.

---

#### 2.5 AI Classification of Responses

- **Overview:** Uses Google Gemini LLM via LangChain integration to classify the WhatsApp reply into one of four categories and generates a JSON response.
- **Nodes Involved:**  
  - Classify Response (AI)  
  - Gemini LLM  
  - Parse AI JSON Output

- **Node Details:**

  - **Classify Response (AI)**  
    - Type: LangChain AI Agent  
    - Role: Sends the WhatsApp reply text to the AI model with instructions to classify the message into: confirmed, declined, human_requested, or invalid_response.  
    - Configuration: Custom prompt with role and instructions, injecting user message dynamically.  
    - Inputs: WhatsApp reply JSON from "Receive WhatsApp Response".  
    - Outputs: Raw AI response text to "Parse AI JSON Output".  
    - Edge Cases: AI misclassification, connectivity issues to AI service.

  - **Gemini LLM**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Underlying large language model used by the agent node to process classification.  
    - Configuration: Model specified as "models/gemini-2.0-flash-lite-001".  
    - Inputs: From "Classify Response (AI)".  
    - Outputs: AI-generated text with classification JSON.

  - **Parse AI JSON Output**  
    - Type: Code (JavaScript)  
    - Role: Parses the AI output text to extract the JSON classification object containing status, reply, and optional suggestion fields.  
    - Configuration: Extracts substring between first and last curly braces, parses JSON safely, handles errors returning fallback invalid_response.  
    - Inputs: AI output string.  
    - Outputs: Structured JSON for routing.  
    - Edge Cases: Malformed AI output, JSON parsing errors.

---

#### 2.6 Response Routing and CRM Update

- **Overview:** Routes the parsed AI classification to send the appropriate WhatsApp reply and update Zoho CRM contact status accordingly.
- **Nodes Involved:**  
  - Route by Classification  
  - Reply: Confirmed  
  - Reply: Declined  
  - Reply: Request Human  
  - Reply: Invalid  
  - Find Confirmed Contact in Zoho  
  - Find Declined Contact in Zoho  
  - Update CRM: Confirmed  
  - Update CRM: Declined  
  - Prepare Owner ID  
  - Assign CRM Owner

- **Node Details:**

  - **Route by Classification**  
    - Type: Switch  
    - Role: Routes flow based on AI classification status: confirmed, declined, human_requested, invalid_response.  
    - Configuration: Matches `$json.status` exactly with one of the four categories.  
    - Inputs: Parsed AI JSON output.  
    - Outputs: Routes to corresponding reply nodes.

  - **Reply: Confirmed**  
    - Type: HTTP Request  
    - Role: Sends Twilio WhatsApp message with AI-generated confirmation reply to lead.  
    - Configuration: Uses Twilio API credentials, dynamically sets From and To from webhook data, body from AI reply.  
    - Inputs: "Route by Classification" confirmed output.  
    - Outputs: Finds confirmed contact in Zoho.  
    - Edge Cases: Messaging failures.

  - **Find Confirmed Contact in Zoho**  
    - Type: HTTP Request  
    - Role: Searches Zoho CRM contact by WhatsApp ID (phone number) to retrieve contact details.  
    - Configuration: OAuth2 authentication, search by phone.  
    - Inputs: Output of "Reply: Confirmed".  
    - Outputs: Passes to "Prepare Owner ID".  
    - Edge Cases: No contact found.

  - **Prepare Owner ID**  
    - Type: Set  
    - Role: Prepares JSON payload setting CRM owner to a fixed ID (e.g., "845610000000531019").  
    - Inputs: Output of "Find Confirmed Contact in Zoho".  
    - Outputs: Passes to "Assign CRM Owner".  
    - Edge Cases: Hardcoded owner ID may need to be configurable.

  - **Assign CRM Owner**  
    - Type: HTTP Request  
    - Role: Updates Zoho CRM contact owner field using a PUT request.  
    - Inputs: Payload from "Prepare Owner ID".  
    - Outputs: Updates contact status via "Update CRM: Confirmed".  
    - Edge Cases: API auth failures.

  - **Update CRM: Confirmed**  
    - Type: Zoho CRM node  
    - Role: Updates contact's custom "Status" field to "Confirmed".  
    - Inputs: Contact ID and data from previous steps.  
    - Outputs: Workflow ends or continues as per design.

  - **Reply: Declined**  
    - Type: HTTP Request  
    - Role: Sends Twilio WhatsApp message with AI-generated decline reply.  
    - Inputs: "Route by Classification" declined output.  
    - Outputs: Searches declined contact in Zoho.  
    - Edge Cases: Messaging or search failures.

  - **Find Declined Contact in Zoho**  
    - Type: HTTP Request  
    - Role: Searches Zoho CRM contact by phone for declined leads.  
    - Inputs: Output of "Reply: Declined".  
    - Outputs: Passes to "Update CRM: Declined".  

  - **Update CRM: Declined**  
    - Type: Zoho CRM node  
    - Role: Updates contact's custom "Status" field to "Declined".  
    - Inputs: Contact ID from search.  
    - Outputs: Ends flow.

  - **Reply: Request Human**  
    - Type: HTTP Request  
    - Role: Sends message to lead indicating human assistance will follow.  
    - Inputs: "Route by Classification" human_requested output.  
    - Outputs: Ends flow.

  - **Reply: Invalid**  
    - Type: HTTP Request  
    - Role: Sends fallback message for unclear or invalid responses.  
    - Inputs: "Route by Classification" invalid_response output.  
    - Outputs: Ends flow.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                               | Input Node(s)                     | Output Node(s)                                  | Sticky Note                                                                                     |
|----------------------------|----------------------------------|-----------------------------------------------|----------------------------------|------------------------------------------------|------------------------------------------------------------------------------------------------|
| Trigger: Schedule Lead Check| Schedule Trigger                 | Periodically triggers lead fetching            | None                             | Fetch FB Leads                                 | Fires periodically to check Facebook Lead Ads for new leads.                                  |
| Fetch FB Leads             | HTTP Request                    | Fetches new leads from Facebook Graph API      | Trigger: Schedule Lead Check     | Extract Lead Info                              | Makes a request to Facebook Graph API to fetch new lead data.                                 |
| Extract Lead Info          | Set                            | Extracts Name, Phone, Email from Facebook lead | Fetch FB Leads                  | Check Zoho Contact Exists                       | Extracts Name, Phone, and Email fields from Facebook lead data.                               |
| Check Zoho Contact Exists  | HTTP Request                   | Searches Zoho CRM for contact by phone         | Extract Lead Info               | If New Lead                                    | Checks Zoho CRM if the contact already exists using phone number.                             |
| If New Lead                | If                             | Routes based on whether contact exists          | Check Zoho Contact Exists       | Create Zoho Contact, Send WhatsApp Confirmation| Routes based on whether the contact already exists in Zoho.                                  |
| Create Zoho Contact        | Zoho CRM                       | Creates new contact with "Pending" status       | If New Lead                    | Send WhatsApp Confirmation                      | Creates a new contact in Zoho CRM with lead info and "Pending" status.                        |
| Send WhatsApp Confirmation | HTTP Request                   | Sends WhatsApp message to confirm interest      | Create Zoho Contact             | None                                           | Sends WhatsApp message asking the lead to confirm interest.                                  |
| Receive WhatsApp Response  | Webhook                        | Receives WhatsApp replies from Twilio           | External                      | Classify Response (AI)                          | Webhook to receive reply messages from Twilio/WhatsApp.                                      |
| Classify Response (AI)     | LangChain AI Agent             | Classifies WhatsApp response using AI            | Receive WhatsApp Response       | Parse AI JSON Output                            | Uses AI to classify user response into confirmed, declined, etc.                             |
| Gemini LLM                 | LangChain Google Gemini Chat   | Underlying LLM model for classification          | Classify Response (AI)          | Classify Response (AI)                          | Google Gemini model used to interpret WhatsApp replies.                                      |
| Parse AI JSON Output       | Code (JavaScript)              | Parses AI output JSON for status and reply       | Classify Response (AI)          | Route by Classification                         | Parses the JSON output from AI Agent to extract status and reply.                            |
| Route by Classification    | Switch                        | Routes flow based on classification status       | Parse AI JSON Output            | Reply: Confirmed, Reply: Declined, Reply: Request Human, Reply: Invalid | Routes based on classification: confirmed, declined, human, or invalid.                      |
| Reply: Confirmed           | HTTP Request                   | Sends confirmation WhatsApp reply                 | Route by Classification        | Find Confirmed Contact in Zoho                  | Sends follow-up message for confirmed leads.                                                 |
| Find Confirmed Contact in Zoho| HTTP Request                | Finds confirmed contact in Zoho CRM               | Reply: Confirmed               | Prepare Owner ID                                | Search Zoho CRM for contact with confirmed phone number.                                    |
| Prepare Owner ID           | Set                            | Prepares owner ID JSON for CRM update             | Find Confirmed Contact in Zoho | Assign CRM Owner                                | Sets the CRM owner ID before updating the record.                                           |
| Assign CRM Owner           | HTTP Request                   | Updates contact owner in Zoho CRM                  | Prepare Owner ID               | Update CRM: Confirmed                           | Updates the contact's owner field in Zoho CRM.                                              |
| Update CRM: Confirmed      | Zoho CRM                      | Updates contact status to "Confirmed"              | Assign CRM Owner               | None                                           | Updates Zoho CRM contact to mark as "Confirmed".                                            |
| Reply: Declined            | HTTP Request                   | Sends decline WhatsApp reply                        | Route by Classification        | Find Declined Contact in Zoho                   | Sends follow-up message for declined responses.                                             |
| Find Declined Contact in Zoho| HTTP Request                 | Finds declined contact in Zoho CRM                  | Reply: Declined               | Update CRM: Declined                            | Search Zoho CRM for declined lead by phone.                                                 |
| Update CRM: Declined       | Zoho CRM                      | Updates contact status to "Declined"                | Find Declined Contact in Zoho  | None                                           | Marks lead in Zoho CRM as "Declined".                                                      |
| Reply: Request Human       | HTTP Request                   | Sends message indicating human assistance requested | Route by Classification        | None                                           | Sends response when lead requests human interaction.                                        |
| Reply: Invalid            | HTTP Request                   | Sends fallback message for invalid replies          | Route by Classification        | None                                           | Sends fallback response for unclear or invalid replies.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "Trigger: Schedule Lead Check"  
   - Set interval trigger to run every few seconds (e.g., every 10 seconds).

2. **Add HTTP Request node to fetch Facebook leads**  
   - Name: "Fetch FB Leads"  
   - Method: GET  
   - URL: `https://graph.facebook.com/v22.0/2056495664847309/leads?access_token={{FACEBOOK_ACCESS_TOKEN}}`  
   - Replace `{{FACEBOOK_ACCESS_TOKEN}}` with a valid Facebook Graph API token.  
   - Connect from Schedule Trigger.

3. **Add Set node to extract lead info**  
   - Name: "Extract Lead Info"  
   - Assign fields:  
     - Name = `{{$json.data[0].field_data[0].values[0]}}`  
     - Phone Number = `{{$json.data[0].field_data[1].values[0]}}`  
     - Email = `{{$json.data[0].field_data[2].values[0]}}`  
   - Connect from Fetch FB Leads.

4. **Add HTTP Request node to check Zoho CRM contact existence**  
   - Name: "Check Zoho Contact Exists"  
   - Method: GET  
   - URL: `https://www.zohoapis.eu/crm/v2/Contacts/search`  
   - Query Parameter: `phone={{$json['Phone Number']}}`  
   - Authentication: OAuth2 (Zoho credentials configured)  
   - Connect from Extract Lead Info.

5. **Add If node to check if lead is new**  
   - Name: "If New Lead"  
   - Condition: Check if `$json.data` is empty or not.  
   - Connect from Check Zoho Contact Exists.

6. **Add Zoho CRM node to create new contact**  
   - Name: "Create Zoho Contact"  
   - Operation: Create contact  
   - Fields:  
     - LastName = `{{$json.Name}}`  
     - Email = `{{$json.Email}}`  
     - Phone = `{{$json['Phone Number']}}`  
     - Custom Field Status = "Pending"  
   - Connect from If New Lead (true branch).

7. **Add HTTP Request node to send WhatsApp confirmation via Twilio**  
   - Name: "Send WhatsApp Confirmation"  
   - Method: POST  
   - URL: `https://api.twilio.com/2010-04-01/Accounts/YOUR_TWILIO_ACCOUNT_SID/Messages.json`  
   - Authentication: HTTP Basic (Twilio Account SID and Auth Token)  
   - Body (multipart/form-data):  
     - From: `whatsapp:+14155238886` (Twilio sandbox number)  
     - To: `whatsapp:{{ $json['Phone Number'] }}`  
     - Body: `Hi {{ $json.Name }}, confirm yes or no`  
   - Connect from Create Zoho Contact.

8. **Add Webhook node to receive WhatsApp replies**  
   - Name: "Receive WhatsApp Response"  
   - HTTP Method: POST  
   - Path: `custrep`  
   - Configure webhook URL in Twilio WhatsApp to point here.

9. **Add LangChain AI Agent node for classification**  
   - Name: "Classify Response (AI)"  
   - Use Google Gemini LLM as the model.  
   - Prompt: Instruct AI to classify the message as one of: confirmed, declined, human_requested, invalid_response, returning JSON with status, reply, suggestion.  
   - Input text: WhatsApp message body from webhook.

10. **Add LangChain Google Gemini Chat node**  
    - Name: "Gemini LLM"  
    - Model: `models/gemini-2.0-flash-lite-001`  
    - Connect as AI service for "Classify Response (AI)".

11. **Add Code node to parse AI JSON output**  
    - Name: "Parse AI JSON Output"  
    - JavaScript code to extract JSON from AI text safely, returning status, reply, suggestion.

12. **Add Switch node "Route by Classification"**  
    - Routes based on `$json.status` with cases: confirmed, declined, human_requested, invalid_response.

13. **For each classification branch, add corresponding HTTP Request node to send reply via Twilio WhatsApp**  
    - Node Names: "Reply: Confirmed", "Reply: Declined", "Reply: Request Human", "Reply: Invalid"  
    - Use Twilio API with correct From and To fields from webhook data, Body from AI reply.

14. **Add HTTP Request node "Find Confirmed Contact in Zoho"**  
    - Searches Zoho CRM by phone from WhatsApp reply for confirmed leads.

15. **Add Set node "Prepare Owner ID"**  
    - Sets JSON with fixed Owner ID for CRM assignment.

16. **Add HTTP Request node "Assign CRM Owner"**  
    - Updates Zoho CRM contact owner field.

17. **Add Zoho CRM node "Update CRM: Confirmed"**  
    - Updates contact status field to "Confirmed".

18. **Add HTTP Request node "Find Declined Contact in Zoho"**  
    - Searches Zoho CRM for declined leads.

19. **Add Zoho CRM node "Update CRM: Declined"**  
    - Updates contact status field to "Declined".

20. **Connect all nodes according to the flow described, ensuring proper error handling and credentials setup (Facebook, Twilio, Zoho OAuth2).**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow integrates Facebook Lead Ads, Twilio WhatsApp API, Google Gemini LLM, and Zoho CRM APIs.  | Workflow description and API usage overview.                                                   |
| Facebook API version used: v22.0.                                                                   | Facebook Graph API documentation: https://developers.facebook.com/docs/graph-api/              |
| Twilio WhatsApp uses sandbox number +14155238886 by default; replace with production numbers as needed. | Twilio WhatsApp API docs: https://www.twilio.com/docs/whatsapp                                 |
| Zoho CRM API v2 with OAuth2 authentication required for contact search and updates.                | Zoho CRM API docs: https://www.zoho.com/crm/developer/docs/api/v2/                             |
| Google Gemini LLM used via LangChain node; requires API credentials and correct model name.        | Google Gemini docs and LangChain integration info (internal or vendor-specific).               |
| Hardcoded Owner ID in CRM update node may require parameterization for flexibility.                | Consider making owner ID a workflow parameter for multi-user assignment.                       |
| Webhook URL must be publicly accessible and configured correctly in Twilio console for WhatsApp replies. | Important for real-time WhatsApp response capture.                                            |

---

*Disclaimer: The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.*