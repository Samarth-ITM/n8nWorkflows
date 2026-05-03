Hostinger Form Lead Capture & Qualification with OpenAI, Beehiiv & Google Sheets

https://n8nworkflows.xyz/workflows/hostinger-form-lead-capture---qualification-with-openai--beehiiv---google-sheets-4575


# Hostinger Form Lead Capture & Qualification with OpenAI, Beehiiv & Google Sheets

### 1. Workflow Overview

This n8n workflow automates the capture and qualification of leads submitted via a Hostinger website form. It processes incoming form submissions received by email, extracts and validates key lead information using OpenAI, records qualified leads into a Google Sheet, and subscribes them to a Beehiiv newsletter.

The workflow is logically divided into four blocks:

- **1.1 Input Reception:** Captures new form submissions by monitoring a Gmail inbox filtered by Hostinger’s email sender.
- **1.2 Lead Extraction & Qualification:** Uses OpenAI to parse the form response text, extract structured fields, and determine lead qualification based on email and website validity.
- **1.3 Newsletter Subscription:** Retrieves Beehiiv publication details and subscribes qualified leads to the newsletter using the Beehiiv API.
- **1.4 Data Storage:** Appends the extracted and qualified lead data into a designated Google Sheets spreadsheet for record-keeping and further analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new incoming emails from Hostinger containing form submissions to trigger the lead capture process automatically.

**Nodes Involved:**  
- New form trigger

**Node Details:**  

- **New form trigger**  
  - **Type & Role:** Gmail Trigger node; listens for new emails matching criteria.  
  - **Configuration:**  
    - Uses Gmail OAuth2 credentials.  
    - Filters emails by sender address: `enterHostingerSender@notifications.hostinger.com`.  
    - Polls inbox every minute.  
  - **Expressions/Variables:** None.  
  - **Input/Output:** No input; outputs email data to next node.  
  - **Version Requirements:** n8n version supporting Gmail Trigger node v1.2.  
  - **Potential Failures:**  
    - Authentication errors if OAuth2 tokens expire or revoke.  
    - Network issues causing polling failures.  
    - Filtering misconfiguration might miss emails or trigger on wrong messages.  
  - **Sub-workflow:** None.

---

#### 1.2 Lead Extraction & Qualification

**Overview:**  
This block uses OpenAI’s GPT model to interpret the raw form submission text and extract structured lead information, including email, company name, website, and qualification status based on defined business rules.

**Nodes Involved:**  
- Extract & Qualify

**Node Details:**  

- **Extract & Qualify**  
  - **Type & Role:** OpenAI node (via n8n’s LangChain integration) for natural language parsing and lead qualification.  
  - **Configuration:**  
    - Model: GPT-4.1-mini.  
    - System prompt instructs model to parse form response text (from incoming email), extract all fields into JSON, and evaluate:  
      - If email domain is free provider → unqualified, reason "Personal email used".  
      - If website missing/invalid → unqualified, reason "Invalid or missing website".  
      - Otherwise qualified.  
    - Output: JSON object containing all form variables plus `isQualified` boolean and `reason` string.  
  - **Expressions/Variables:** Uses incoming email text as input: `{{ $json.text }}`.  
  - **Input/Output:** Input from Gmail trigger; outputs structured JSON with qualification results.  
  - **Version Requirements:** LangChain OpenAI node compatible with GPT-4.1-mini model, n8n version supporting node v1.8.  
  - **Potential Failures:**  
    - API key/authentication errors with OpenAI.  
    - Timeout or rate limits from OpenAI API.  
    - Unexpected input format causing parsing errors or incomplete JSON output.  
  - **Sub-workflow:** None.

---

#### 1.3 Newsletter Subscription

**Overview:**  
This block synchronizes qualified leads into a Beehiiv newsletter by first retrieving the list of Beehiiv publications, then adding the lead as a subscriber to the first publication.

**Nodes Involved:**  
- List Beehiiv publications  
- Add Beehiiv subscriber

**Node Details:**  

- **List Beehiiv publications**  
  - **Type & Role:** HTTP Request node to call Beehiiv API to fetch publications.  
  - **Configuration:**  
    - GET request to `https://api.beehiiv.com/v2/publications`.  
    - Uses HTTP Header Authentication with Beehiiv newsletter API key.  
  - **Input/Output:** Receives input from Extract & Qualify; outputs list of publications (JSON).  
  - **Potential Failures:**  
    - Auth errors if API key invalid.  
    - Network issues or API downtime.  
  - **Sub-workflow:** None.

- **Add Beehiiv subscriber**  
  - **Type & Role:** HTTP Request node to add a subscriber to a specific Beehiiv publication.  
  - **Configuration:**  
    - POST request to `https://api.beehiiv.com/v2/publications/{{ $json.data[0].id }}/subscriptions`.  
    - Body includes subscriber email extracted from `Extract & Qualify` node JSON output.  
    - Uses same Beehiiv HTTP Header Authentication credentials.  
  - **Expressions/Variables:** Email value: `={{ $('Extract & Qualify').item.json.message.content.email }}`  
  - **Input/Output:** Input from List Beehiiv publications; outputs subscription response.  
  - **Potential Failures:**  
    - Invalid publication ID if publications list empty or changed.  
    - Duplicate subscriber errors if email already subscribed.  
    - Auth or network errors.  
  - **Sub-workflow:** None.

---

#### 1.4 Data Storage

**Overview:**  
This block appends all extracted and qualified lead information into a Google Sheet for record-keeping and further review.

**Nodes Involved:**  
- insert in Sheets

**Node Details:**  

- **insert in Sheets**  
  - **Type & Role:** Google Sheets node to append data rows.  
  - **Configuration:**  
    - Document ID and sheet name configured for a specific Google Sheet document and worksheet.  
    - Data mapping: multiple columns set with expressions pulling data from `Extract & Qualify` node’s JSON output fields such as Name, Role, Company, Email, Website, Budget, Reason, Is Qualified, etc.  
    - Append mode enabled to add new rows.  
    - Uses Google OAuth2 credentials.  
  - **Expressions/Variables:** Multiple expressions like `={{ $('Extract & Qualify').item.json.message.content.name }}` for each column.  
  - **Input/Output:** Input from Add Beehiiv subscriber node; no output connection.  
  - **Potential Failures:**  
    - Authentication errors with Google OAuth2.  
    - Sheet ID or name misconfiguration causing append failure.  
    - Data type mismatches or empty data fields.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                                                                                                                                                  |
|-------------------------|---------------------------|-------------------------------|------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| New form trigger        | Gmail Trigger             | Capture new Hostinger form email | None                   | Extract & Qualify             | Create an account on [Hostinger](https://hostinger.es?REFERRALCODE=6MKHELLOUQOS); set up a website form and test; check email for Hostinger sender address.                                                               |
| Extract & Qualify       | OpenAI (LangChain)        | Parse form response, extract & qualify lead | New form trigger       | List Beehiiv publications     | Extract form response: Hostinger does not store form data; OpenAI extracts lead info as JSON.                                                                                                                               |
| List Beehiiv publications | HTTP Request             | Retrieve Beehiiv publications list | Extract & Qualify      | Add Beehiiv subscriber        | Newsletter sync: adds form submitters to Beehiiv newsletter; ensure inclusion in terms and conditions; Beehiiv chosen for strong free features. [Beehiiv](https://www.beehiiv.com?via=1node-ai)                            |
| Add Beehiiv subscriber  | HTTP Request              | Subscribe lead to Beehiiv newsletter | List Beehiiv publications | insert in Sheets              | Newsletter sync note (same as above).                                                                                                                                                                                       |
| insert in Sheets        | Google Sheets             | Append lead data to Google Sheets | Add Beehiiv subscriber | None                         |                                                                                                                                                                                                                              |
| Sticky Note             | Sticky Note               | Setup instructions             | None                   | None                         | See above for Hostinger setup steps.                                                                                                                                                                                       |
| Sticky Note1            | Sticky Note               | Newsletter sync explanation    | None                   | None                         | See above for newsletter sync details.                                                                                                                                                                                     |
| Sticky Note2            | Sticky Note               | Extract form response note     | None                   | None                         | See above for extraction explanation.                                                                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Credential: Gmail OAuth2 (set up with appropriate scopes)  
   - Filter: Sender = `enterHostingerSender@notifications.hostinger.com`  
   - Poll interval: Every minute  
   - No input connections.

2. **Create OpenAI node (LangChain):**  
   - Type: OpenAI (LangChain integration)  
   - Credential: OpenAI API key with proper access  
   - Model: GPT-4.1-mini  
   - Input: Use incoming email text from Gmail Trigger node (`{{ $json.text }}`)  
   - System prompt:  
     ```
     You are a lead qualification engine. You will receive the text of an email containing a form response.

     Here is the form response: 
     {{ $json.text }}

     Your task is to evaluate the fields: "email", "company name", and "website" to determine if a lead is qualified, as well as extracting the rest of the variables in the form response to output a JSON with all the fields.

     "isQualified" → true or false
     "reason" → short explanation if unqualified, empty string if qualified

     Rules:

     If the email domain is from a free provider (e.g., gmail.com, yahoo.com, hotmail.com, outlook.com), set isQualified = false and set reason = "Personal email used".

     If the website field is missing, invalid, or not a proper URL, set isQualified = false and set reason = "Invalid or missing website".

     If both email and website are acceptable, set isQualified = true and set reason = "".

     Response format: Only return JSON with all the variables. No explanations, no additional text.
     ```
   - Enable JSON output parsing.  
   - Connect Gmail Trigger node output to this node.

3. **Create HTTP Request node to list Beehiiv publications:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.beehiiv.com/v2/publications`  
   - Authentication: HTTP Header Auth with Beehiiv API key (set up credentials)  
   - Connect OpenAI output to this node.

4. **Create HTTP Request node to add Beehiiv subscriber:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.beehiiv.com/v2/publications/{{ $json.data[0].id }}/subscriptions` (dynamic publication ID from previous node)  
   - Authentication: Same Beehiiv HTTP Header Auth credentials  
   - Body Parameters:  
     - email: `={{ $('Extract & Qualify').item.json.message.content.email }}`  
   - Connect List Beehiiv publications node output to this node.

5. **Create Google Sheets node to append data:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheet document ID  
   - Sheet Name: e.g., `gid=0` or sheet tab name  
   - Columns: Map the following fields from OpenAI output (`Extract & Qualify`) JSON content:  
     - Name, Role, Size, Email, Budget, Reason, Company, Message, Website, Services, Is Qualified  
   - Authentication: Google OAuth2 credentials with Sheets API access  
   - Connect Add Beehiiv subscriber node output to this node.

6. **Connect nodes in order:**  
   - Gmail Trigger → Extract & Qualify (OpenAI) → List Beehiiv publications → Add Beehiiv subscriber → insert in Sheets

7. **Optional:**  
   - Add sticky notes in n8n editor with setup instructions and context for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                               | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create an account on Hostinger to use their form services and test email triggers.                                                                                                                         | https://hostinger.es?REFERRALCODE=6MKHELLOUQOS                                                    |
| Beehiiv newsletter integration requires adding subscribers via their API; ensure compliance with data privacy by including newsletter subscription in your terms and conditions. Beehiiv offers strong free-tier features. | https://www.beehiiv.com?via=1node-ai                                                              |
| Hostinger does not store form submissions; OpenAI is used to extract structured JSON data from email text responses.                                                                                      | —                                                                                                 |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.