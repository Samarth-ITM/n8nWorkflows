Multi-Channel Campaign Messaging with GPT-4 and Salesforce

https://n8nworkflows.xyz/workflows/multi-channel-campaign-messaging-with-gpt-4-and-salesforce-6219


# Multi-Channel Campaign Messaging with GPT-4 and Salesforce

### 1. Workflow Overview

This workflow automates multi-channel campaign messaging by integrating Salesforce campaign data with GPT-4 AI message generation and sending personalized outbound communications via SMS, Email, or WhatsApp. It targets marketing and sales teams seeking to efficiently reach campaign members with tailored messages based on campaign details and contact preferences.

**Logical Blocks:**

- **1.1 Scheduled Campaign Retrieval:** Periodically triggers the workflow to fetch active Salesforce campaigns requiring message processing.
- **1.2 Campaign Members Fetch:** Retrieves members associated with each campaign, including their contact methods and details.
- **1.3 AI Message Generation:** Uses OpenAI's GPT-4 to generate personalized outbound messages tailored to each member’s preferred communication channel and campaign context.
- **1.4 Communication Channel Routing:** Routes messages to appropriate sending nodes (SMS, Email, WhatsApp) based on AI-determined contact method.
- **1.5 Message Sending:** Sends outbound messages through Twilio for SMS and WhatsApp or SMTP for Email.
- **1.6 Post-Send Processing:** Updates Salesforce campaign member records to mark them as processed after message sending.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Campaign Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule and fetches Salesforce campaigns that need processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch Campaign  
  - Sticky Note (Fetch Campaigns)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node for periodic execution  
    - Configuration: Default interval (runs once every minute/hour/day depending on setup)  
    - Inputs: None (trigger)  
    - Outputs: Campaign fetch  
    - Edge cases: Missed runs if n8n server down, scheduling misconfiguration

  - **Fetch Campaign**  
    - Type: Salesforce node (search resource)  
    - Configuration: SOQL query `SELECT Id, Description, Name FROM Campaign` to retrieve campaigns  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Campaign records for downstream processing  
    - Edge cases: Salesforce API limits, authentication failures, empty results if no campaigns match criteria

  - **Sticky Note (Fetch Campaigns)**  
    - Content: "Retrieve campaigns that still require processing."  
    - Role: Documentation only, no functional effect

---

#### 1.2 Campaign Members Fetch

- **Overview:**  
  For each campaign fetched, this block retrieves associated campaign members with their contact info and communication preferences.

- **Nodes Involved:**  
  - Fetch Campaign Members  
  - Sticky Note1 (Fetch Campaign Members)

- **Node Details:**

  - **Fetch Campaign Members**  
    - Type: Salesforce node (search resource)  
    - Configuration: SOQL query:  
      `SELECT Id, CampaignId, Contact.ContactMethod__c, Email, Phone FROM CampaignMember WHERE CampaignId = '{{ $json.Id }}'`  
    - Uses dynamic campaign Id from previous node  
    - Inputs: Campaign data from Fetch Campaign  
    - Outputs: Campaign member records with contact details  
    - Edge cases: Salesforce API limits, authentication errors, missing or null contact fields

  - **Sticky Note1 (Fetch Campaign Members)**  
    - Content: "Retrieve all members associated with the campaigns returned in the previous step. Make the specific query ID for your Campaign."  
    - Role: Documentation

---

#### 1.3 AI Message Generation

- **Overview:**  
  Uses OpenAI GPT-4 model to generate personalized outbound messages for each campaign member based on contact method and campaign description. Returns a strictly formatted JSON object with message details.

- **Nodes Involved:**  
  - OpenAI  
  - Sticky Note2 (Generate Personalized Messages)

- **Node Details:**

  - **OpenAI**  
    - Type: Langchain OpenAI node  
    - Model: GPT-4.1  
    - Configuration:  
      - System prompt instructs the AI to act as a marketing communication specialist generating outbound messages.  
      - Input message includes contact method, phone, email, and campaign description dynamically injected from Salesforce data and previous nodes.  
      - Output: JSON object with keys: method, from, to, subject (email only), message  
    - Inputs: Campaign member data from Fetch Campaign Members and campaign description from Fetch Campaign  
    - Outputs: AI-generated message JSON  
    - Edge cases: API rate limits, prompt formatting errors, unexpected AI output formatting, credential issues  
    - Version: Uses n8n Langchain OpenAI node v1.8

  - **Sticky Note2 (Generate Personalized Messages)**  
    - Content: "Use OpenAI to convert each campaign’s description into tailored messages for every member, selecting the appropriate channel (SMS, Email, WhatsApp, or Telegram)."  
    - Role: Documentation

---

#### 1.4 Communication Channel Routing

- **Overview:**  
  Routes the AI-generated message to the appropriate sending node based on the selected communication method.

- **Nodes Involved:**  
  - Communication Method Switch  
  - Sticky Note3 (Communication Method Mapping)

- **Node Details:**

  - **Communication Method Switch**  
    - Type: Switch node  
    - Configuration:  
      - Evaluates `{{ $json.message.content.method }}` (method returned by OpenAI)  
      - Routes based on string values: "SMS", "Email", "Whatsapp", "Telegram"  
    - Inputs: AI-generated message JSON from OpenAI node  
    - Outputs: Connects separately to Send SMS, Send Email, Send Whatsapp, and directly to HTTP Request (for Telegram or fallback)  
    - Edge cases: Unexpected or null method values, routing errors  
    - Notes: Telegram is present in the switch but no sending node configured; messages routed to HTTP Request node (likely for future extension)

  - **Sticky Note3 (Communication Method Mapping)**  
    - Content: "Assign channel codes as follows: SMS = 0, Email = 1, WhatsApp = 2, Telegram = 3. Ask ChatGPT"  
    - Role: Documentation and reminder

---

#### 1.5 Message Sending

- **Overview:**  
  Sends the generated message through the appropriate channel: SMS and WhatsApp via Twilio, Email via SMTP.

- **Nodes Involved:**  
  - Send SMS  
  - Send Email  
  - Send Whatsapp  
  - Sticky Note (Send SMS)  
  - Sticky Note (Send Whatsapp) [merged with Send SMS notes]

- **Node Details:**

  - **Send SMS**  
    - Type: Twilio node  
    - Configuration:  
      - `to`: dynamic phone number from AI message JSON  
      - `from`: fixed Twilio phone number (e.g., +1xxxxxxxxxx)  
      - `message`: dynamic content text  
    - Inputs: Routed from Communication Method Switch  
    - Outputs: HTTP Request node for Salesforce update  
    - Credentials: Twilio API credentials required  
    - Edge cases: Twilio API errors, invalid phone numbers, message length limits  
    - Notes: Sticky note "Send SMS using Twilio"

  - **Send Email**  
    - Type: Email Send node  
    - Configuration:  
      - `toEmail`: dynamic recipient email  
      - `fromEmail`: static sender email (youremail@yourdomain.com)  
      - `subject`: dynamic from AI JSON  
      - `html`: dynamic email body  
    - Inputs: Routed from Communication Method Switch  
    - Outputs: HTTP Request node for Salesforce update  
    - Credentials: SMTP credentials required  
    - Edge cases: SMTP errors, invalid email addresses, email formatting issues

  - **Send Whatsapp**  
    - Type: Twilio node with WhatsApp enabled  
    - Configuration:  
      - `toWhatsapp`: dynamic phone number  
      - `from`: fixed Twilio phone number  
      - `message`: dynamic content text  
    - Inputs: Routed from Communication Method Switch  
    - Outputs: HTTP Request node for Salesforce update  
    - Credentials: Twilio API credentials required  
    - Edge cases: Twilio WhatsApp API limits, invalid numbers, unsupported regions

---

#### 1.6 Post-Send Processing

- **Overview:**  
  Marks each campaign member as processed in Salesforce to avoid duplicate messaging.

- **Nodes Involved:**  
  - HTTP Request (Salesforce PATCH)  
  - Sticky Note4 (Mark Campaign Members as Processed)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Configuration:  
      - Method: PATCH  
      - URL: Salesforce REST API endpoint to update CampaignMember record with `Processed__c = true`  
      - Dynamic URL uses campaign member Id from previous node outputs  
      - Uses Salesforce OAuth2 credentials  
    - Inputs: From all sending nodes (Send SMS, Send Email, Send Whatsapp) and also directly from Communication Method Switch (for Telegram/fallback)  
    - Outputs: None (terminal)  
    - Edge cases: Salesforce API errors, authorization failures, network timeouts

  - **Sticky Note4 (Mark Campaign Members as Processed)**  
    - Content: "Update each CampaignMember record to indicate that the personalized message has been generated and the member has been processed."  
    - Role: Documentation

---

### 3. Summary Table

| Node Name                   | Node Type                    | Functional Role                            | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                                           |
|-----------------------------|------------------------------|--------------------------------------------|---------------------------|------------------------------------|------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger              | Periodic workflow start                     | None                      | Fetch Campaign                     |                                                                                                      |
| Fetch Campaign             | Salesforce                   | Retrieve campaigns for processing           | Schedule Trigger           | Fetch Campaign Members             |                                                                                                      |
| Sticky Note                | Sticky Note                  | Documentation for Fetch Campaign            | None                      | None                             | Retrieve campaigns that still require processing.                                                     |
| Fetch Campaign Members     | Salesforce                   | Retrieve campaign members for each campaign | Fetch Campaign             | OpenAI                           |                                                                                                      |
| Sticky Note1               | Sticky Note                  | Documentation for Fetch Campaign Members    | None                      | None                             | Retrieve all members associated with the campaigns returned in the previous step.                     |
| OpenAI                    | Langchain OpenAI              | Generate personalized messages              | Fetch Campaign Members     | Communication Method Switch        | Use OpenAI to convert each campaign’s description into tailored messages for every member.           |
| Sticky Note2               | Sticky Note                  | Documentation for AI Message Generation      | None                      | None                             |                                                                                                      |
| Communication Method Switch| Switch                      | Route messages based on contact method       | OpenAI                     | Send SMS, Send Email, Send Whatsapp, HTTP Request | Assign channel codes as follows: SMS = 0, Email = 1, WhatsApp = 2, Telegram = 3.                       |
| Send SMS                   | Twilio                      | Send SMS messages                            | Communication Method Switch| HTTP Request                      | Send SMS using Twilio                                                                                  |
| Send Email                 | Email Send                  | Send Email messages                          | Communication Method Switch| HTTP Request                      |                                                                                                      |
| Send Whatsapp              | Twilio                      | Send WhatsApp messages                       | Communication Method Switch| HTTP Request                      | Send SMS using Twilio                                                                                  |
| HTTP Request               | HTTP Request                | Update CampaignMember record as processed   | Send SMS, Send Email, Send Whatsapp, Communication Method Switch (Telegram fallback) | None                             | Mark Campaign Members as Processed                                                                   |
| Sticky Note4               | Sticky Note                  | Documentation for post-send processing       | None                      | None                             | Update each CampaignMember record to indicate that the personalized message has been generated and processed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set desired interval (e.g., daily or hourly trigger)  
   - No credentials needed

2. **Create Salesforce node "Fetch Campaign":**  
   - Type: Salesforce  
   - Resource: Search  
   - Query: `SELECT Id, Description, Name FROM Campaign`  
   - Connect input from Schedule Trigger node  
   - Configure Salesforce OAuth2 credentials

3. **Create Salesforce node "Fetch Campaign Members":**  
   - Type: Salesforce  
   - Resource: Search  
   - Query: `SELECT Id, CampaignId, Contact.ContactMethod__c, Email, Phone FROM CampaignMember WHERE CampaignId = '{{ $json.Id }}'`  
   - Connect input from Fetch Campaign node  
   - Configure Salesforce OAuth2 credentials

4. **Create OpenAI node "OpenAI":**  
   - Type: Langchain OpenAI  
   - Model: GPT-4.1  
   - Messages Configuration:  
     - System message instructs AI on message generation rules and JSON output format (refer to overview for full prompt content)  
     - Assistant message includes dynamic JSON data with contact method, phone, email, and campaign description injected from previous nodes:  
       ```
       {
         "contactMethod": "{{ $json.Contact.ContactMethod__c }}",
         "phone": "{{ $json.Phone }}",
         "email": "{{ $json.Email }}",
         "campaignDescription": "{{ $('Fetch Campaign').item.json.Description }}"
       }
       ```  
   - Enable JSON output parsing  
   - Connect input from Fetch Campaign Members node  
   - Configure OpenAI API credentials

5. **Create Switch node "Communication Method Switch":**  
   - Type: Switch  
   - Expression: `{{ $json.message.content.method }}`  
   - Rules:  
     - Value "SMS" → Output 0  
     - Value "Email" → Output 1  
     - Value "Whatsapp" → Output 2  
     - Value "Telegram" → Output 3  
   - Connect input from OpenAI node

6. **Create Twilio node "Send SMS":**  
   - Type: Twilio  
   - Parameters:  
     - To: `{{ $json.message.content.to }}`  
     - From: Your Twilio phone number (e.g., +1xxxxxxxxxx)  
     - Message: `{{ $json.message.content.message }}`  
   - Connect input from Communication Method Switch output 0 (SMS)  
   - Add Twilio API credentials

7. **Create Email Send node "Send Email":**  
   - Type: Email Send  
   - Parameters:  
     - To Email: `{{ $json.message.content.to }}`  
     - From Email: your verified sender email (e.g., youremail@yourdomain.com)  
     - Subject: `{{ $json.message.content.subject }}`  
     - HTML Body: `{{ $json.message.content.message }}`  
   - Connect input from Communication Method Switch output 1 (Email)  
   - Add SMTP credentials

8. **Create Twilio node "Send Whatsapp":**  
   - Type: Twilio  
   - Parameters:  
     - To WhatsApp: `{{ $json.message.content.to }}`  
     - From: Your Twilio WhatsApp-enabled phone number  
     - Message: `{{ $json.message.content.message }}`  
   - Enable WhatsApp option in Twilio node  
   - Connect input from Communication Method Switch output 2 (Whatsapp)  
   - Add Twilio API credentials

9. **Create HTTP Request node "Update Campaign Member":**  
   - Type: HTTP Request  
   - Method: PATCH  
   - URL: `https://<your_salesforce_instance>.my.salesforce.com/services/data/v60.0/sobjects/CampaignMember/{{ $json.Id }}`  
   - Body: JSON `{ "Processed__c": true }`  
   - Authentication: Use Salesforce OAuth2 credentials  
   - Connect inputs from Send SMS, Send Email, Send Whatsapp nodes, and Communication Method Switch output 3 (Telegram/fallback)

10. **Add Sticky Notes (optional):**  
    - Add descriptive sticky notes to document each block for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses Twilio to send SMS and WhatsApp messages.                                                  | Twilio official docs: https://www.twilio.com/docs                                               |
| OpenAI GPT-4 is used for AI-driven personalized message generation with a strict JSON output format.     | OpenAI API docs: https://platform.openai.com/docs                                              |
| Salesforce OAuth2 API integration requires proper connected app setup and token management.              | Salesforce API docs: https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/  |
| Email sending requires SMTP credentials with verified sender address.                                    | SMTP setup depends on your email provider                                                      |
| The workflow assumes the Salesforce CampaignMember object has a custom boolean field `Processed__c`.     | Create this field in Salesforce to track processed members                                      |
| Telegram is recognized as a contact method but no sending node is configured; HTTP Request is fallback.  | Extend Telegram support by implementing a Telegram API node or webhook integration if desired  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow designed within compliance of applicable content policies. It contains no illegal, offensive, or protected content. All data manipulated is legal and public.